# Part 93: Cost Optimization for Spring Boot on Cloud
## ขั้นตอนที่ 3321-3360

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้เทคนิคการลดต้นทุน cloud สำหรับ Spring Boot applications รวมถึง JVM right-sizing, Spot instances, Reserved instances, Query optimization, Connection pool tuning, CDN caching และ Cost monitoring

---

## ขั้นตอนที่ 3321: Right-Sizing JVM Heap ใน Containers

การตั้งค่า JVM heap ที่ไม่เหมาะสมเป็นสาเหตุหลักของค่าใช้จ่ายที่สูงเกินความจำเป็น

### วิเคราะห์ Heap Usage

```bash
# ดู heap usage จริงๆ
kubectl exec -it pod/shophub-api-xxx -- jcmd 1 GC.heap_info

# หรือใช้ jstat
kubectl exec -it pod/shophub-api-xxx -- jstat -gc 1 5000

# ดู actual memory usage
kubectl top pod shophub-api-xxx --containers
```

### Heap Sizing Formula

```
JVM Heap = Container Memory × 0.75
Off-heap (Metaspace, CodeCache, Thread stacks) = Container Memory × 0.25

ตัวอย่าง:
- Container limit: 1GB
- JVM heap max: 768MB (-Xmx768m หรือ -XX:MaxRAMPercentage=75)
- Off-heap: 256MB

ถ้า app ใช้ heap เฉลี่ย 400MB:
- Container memory เหมาะสม = 600MB (400MB / 0.75 + buffer)
- ไม่ต้องใช้ 1GB container → ประหยัด 40%!
```

### Memory Analysis Script

```java
// monitoring/MemoryAnalysisService.java
@Service
public class MemoryAnalysisService {
    
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedRate = 60000)
    public void recordMemoryMetrics() {
        MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
        
        MemoryUsage heapUsage = memoryMXBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryMXBean.getNonHeapMemoryUsage();
        
        // Heap
        double heapUsedMB = heapUsage.getUsed() / 1024.0 / 1024.0;
        double heapMaxMB = heapUsage.getMax() / 1024.0 / 1024.0;
        double heapUtilization = heapUsage.getUsed() * 100.0 / heapUsage.getMax();
        
        meterRegistry.gauge("jvm.heap.used.mb", heapUsedMB);
        meterRegistry.gauge("jvm.heap.max.mb", heapMaxMB);
        meterRegistry.gauge("jvm.heap.utilization.percent", heapUtilization);
        
        // Alert ถ้า heap utilization สูงมาก
        if (heapUtilization > 85) {
            log.warn("Heap utilization สูงมาก: {}% (used: {}MB / max: {}MB)",
                String.format("%.1f", heapUtilization),
                String.format("%.0f", heapUsedMB),
                String.format("%.0f", heapMaxMB));
        }
        
        // แนะนำ right-sizing
        if (heapUtilization < 30) {
            log.info("Heap ว่างมาก ({:.1f}%) - พิจารณาลด container memory", heapUtilization);
        }
    }
    
    public MemoryReport generateReport() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        Runtime runtime = Runtime.getRuntime();
        
        return MemoryReport.builder()
            .heapUsedMB(memBean.getHeapMemoryUsage().getUsed() / 1024 / 1024)
            .heapMaxMB(memBean.getHeapMemoryUsage().getMax() / 1024 / 1024)
            .nonHeapUsedMB(memBean.getNonHeapMemoryUsage().getUsed() / 1024 / 1024)
            .availableProcessors(runtime.availableProcessors())
            .recommendation(calculateSizingRecommendation())
            .build();
    }
    
    private String calculateSizingRecommendation() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        long heapUsed = memBean.getHeapMemoryUsage().getUsed();
        long heapMax = memBean.getHeapMemoryUsage().getMax();
        double utilization = (double) heapUsed / heapMax;
        
        if (utilization < 0.3) {
            long recommendedHeap = (long) (heapUsed * 1.5); // 50% buffer
            return String.format("ลด heap เป็น %dMB (ประหยัดได้ ~%d%%)",
                recommendedHeap / 1024 / 1024,
                (int) ((heapMax - recommendedHeap) * 100 / heapMax));
        } else if (utilization > 0.85) {
            long recommendedHeap = (long) (heapMax * 1.3); // เพิ่ม 30%
            return String.format("เพิ่ม heap เป็น %dMB เพื่อป้องกัน OOM",
                recommendedHeap / 1024 / 1024);
        }
        
        return "Heap sizing เหมาะสมแล้ว";
    }
}
```

---

## ขั้นตอนที่ 3322: Spot Instances สำหรับ Batch/Async Workloads

Spot instances ราคาถูกกว่า On-demand 70-90% แต่อาจถูก interrupt ได้

### Architecture สำหรับ Spot-tolerant Workloads

```yaml
# k8s/batch-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shophub-batch-processor
spec:
  replicas: 3
  template:
    spec:
      # ใช้ Spot instances สำหรับ batch
      nodeSelector:
        node-type: spot
      tolerations:
      - key: "spot-instance"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      terminationGracePeriodSeconds: 120  # ให้เวลา 2 นาทีก่อน terminate
      containers:
      - name: batch-processor
        image: shophub/batch-processor:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "curl -X POST localhost:8080/actuator/shutdown && sleep 30"]
```

### Graceful Shutdown สำหรับ Spot Interruption

```java
// shutdown/SpotInterruptionHandler.java
@Component
@Slf4j
public class SpotInterruptionHandler {
    
    private final JobService jobService;
    private final MessageQueueService queueService;
    private volatile boolean shutdownRequested = false;
    
    // AWS Spot ส่ง SIGTERM 2 นาทีก่อน terminate
    @EventListener(ContextClosingEvent.class)
    public void handleShutdown() {
        log.warn("Spot instance กำลังถูก terminate - กำลัง graceful shutdown...");
        shutdownRequested = true;
        
        // 1. หยุดรับ jobs ใหม่
        jobService.pauseNewJobAcceptance();
        
        // 2. ส่ง in-progress jobs กลับไปที่ queue
        List<Job> inProgressJobs = jobService.getInProgressJobs();
        inProgressJobs.forEach(job -> {
            queueService.requeueJob(job.getId());
            log.info("ส่ง job {} กลับ queue สำเร็จ", job.getId());
        });
        
        // 3. รอ jobs ที่กำลังจบ (max 90 วินาที)
        long deadline = System.currentTimeMillis() + 90_000;
        while (System.currentTimeMillis() < deadline && jobService.hasActiveJobs()) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        
        log.info("Graceful shutdown เสร็จสิ้น");
    }
    
    public boolean isShutdownRequested() {
        return shutdownRequested;
    }
}

// batch/BatchJobProcessor.java
@Component
public class BatchJobProcessor {
    
    private final SpotInterruptionHandler interruptionHandler;
    
    @Scheduled(fixedDelay = 1000)
    public void processBatch() {
        // ตรวจสอบก่อน process
        if (interruptionHandler.isShutdownRequested()) {
            return; // หยุด process เมื่อมีการ shutdown
        }
        
        // Process job...
    }
}
```

### EKS Karpenter สำหรับ Auto Spot Management

```yaml
# karpenter/spot-nodepool.yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]  # Fallback to on-demand
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      nodeClassRef:
        name: default
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
  limits:
    cpu: 1000
    memory: 1000Gi
```

---

## ขั้นตอนที่ 3323: Reserved Instances สำหรับ Base Load

```
การคำนวณ Reserved Instances:

1. วัด baseline load (load ที่มีตลอด 24/7)
   - ถ้า production มีอยู่ตลอด 10 pods × t3.medium
   - แต่ minimum ที่ใช้ตลอดเวลา = 6 pods
   
2. Reserve 6 instances (1-year, no upfront)
   - On-demand: $0.0416/hour × 6 × 8760 hours = $2,188/year
   - Reserved 1yr: $0.0264/hour × 6 × 8760 hours = $1,387/year
   - ประหยัด: $801/year (37%)
   
3. ส่วนที่เกิน baseline ใช้ Spot
   - 4 remaining pods → Spot (ประหยัด 70-90% เพิ่มเติม)
```

### Cost Tracking Dashboard

```java
// cost/CostTrackingService.java
@Service
public class CostTrackingService {
    
    @Value("${aws.cost.on-demand-rate-per-vcpu-hour:0.04}")
    private BigDecimal onDemandRatePerVcpuHour;
    
    @Value("${aws.cost.spot-savings-percent:70}")
    private int spotSavingsPercent;
    
    public CostReport generateDailyCostReport() {
        int cpuCores = Runtime.getRuntime().availableProcessors();
        int replicaCount = kubernetesService.getReplicaCount("shophub-api");
        
        BigDecimal dailyHours = new BigDecimal("24");
        
        // ต้นทุนถ้าใช้ On-demand ทั้งหมด
        BigDecimal onDemandCost = onDemandRatePerVcpuHour
            .multiply(new BigDecimal(cpuCores * replicaCount))
            .multiply(dailyHours);
        
        // ต้นทุนจริงพร้อม Reserved + Spot
        BigDecimal actualCost = calculateActualCost(cpuCores, replicaCount);
        
        BigDecimal savings = onDemandCost.subtract(actualCost);
        BigDecimal savingsPercent = savings.divide(onDemandCost, 2, RoundingMode.HALF_UP)
            .multiply(new BigDecimal("100"));
        
        return CostReport.builder()
            .date(LocalDate.now())
            .onDemandEquivalent(onDemandCost)
            .actualCost(actualCost)
            .savings(savings)
            .savingsPercent(savingsPercent)
            .build();
    }
}
```

---

## ขั้นตอนที่ 3324: Reducing Unnecessary DB Queries

Query ที่ไม่จำเป็นเป็นต้นทุนซ่อนเร้นที่ใหญ่มาก

### N+1 Query Problem Detection

```java
// config/QueryCountingConfig.java
@Configuration
@Profile({"dev", "staging"})
public class QueryCountingConfig {
    
    @Bean
    public DataSource queryCountingDataSource(DataSource dataSource) {
        // Wrap datasource ด้วย P6Spy หรือ Datasource-proxy
        return ProxyDataSourceBuilder.create(dataSource)
            .name("QueryCounter")
            .countQuery() // นับ queries
            .logSlowQueryBySlf4j(300, TimeUnit.MILLISECONDS) // Log slow queries > 300ms
            .listener(new QueryCountingListener()) // Custom listener
            .build();
    }
}

// listener/QueryCountingListener.java
public class QueryCountingListener extends QueryExecutionListenerAdapter {
    
    private static final ThreadLocal<Integer> queryCount = ThreadLocal.withInitial(() -> 0);
    
    @Override
    public void beforeQuery(ExecutionInfo execInfo, List<QueryInfo> queryInfoList) {
        queryCount.set(queryCount.get() + 1);
    }
    
    @Override
    public void afterMethod(MethodExecutionContext executionContext) {
        // Reset after each request
        queryCount.remove();
    }
    
    public static int getQueryCount() {
        return queryCount.get();
    }
}

// interceptor/QueryCountInterceptor.java
@Component
@Profile({"dev", "staging"})
public class QueryCountInterceptor implements HandlerInterceptor {
    
    private final AlertService alertService;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                              Object handler) {
        QueryCountingListener.reset();
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        int count = QueryCountingListener.getQueryCount();
        
        if (count > 20) {
            log.warn("N+1 query detected! {} queries for {} {}",
                count, request.getMethod(), request.getRequestURI());
            
            alertService.sendN1Alert(request.getRequestURI(), count);
        }
        
        // Add query count header for debugging
        response.addHeader("X-Query-Count", String.valueOf(count));
    }
}
```

### Fix N+1 ด้วย JPQL JOIN FETCH

```java
// repository/OrderRepository.java
@Repository
public interface OrderRepository extends JpaRepository<Order, String> {
    
    // BAD: N+1 problem - load orders แล้ว lazy load items ทีละ order
    List<Order> findByCustomerId(String customerId);
    
    // GOOD: JOIN FETCH - load ทุกอย่างใน 1 query
    @Query("SELECT DISTINCT o FROM Order o " +
           "LEFT JOIN FETCH o.items i " +
           "LEFT JOIN FETCH i.product " +
           "WHERE o.customerId = :customerId")
    List<Order> findByCustomerIdWithItems(@Param("customerId") String customerId);
    
    // ใช้ EntityGraph สำหรับ flexible loading
    @EntityGraph(attributePaths = {"items", "items.product", "shippingAddress"})
    List<Order> findByStatus(OrderStatus status);
}
```

### Query Analysis Tool

```java
// service/QueryAnalysisService.java
@Service
@Slf4j
public class QueryAnalysisService {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public QueryPlan analyzeQuery(String jpql) {
        // ดู execution plan ของ query
        Query query = entityManager.createQuery(jpql);
        
        // PostgreSQL EXPLAIN ANALYZE
        String sql = extractSql(query);
        List<Object[]> plan = entityManager
            .createNativeQuery("EXPLAIN ANALYZE " + sql)
            .getResultList();
        
        return QueryPlan.builder()
            .jpql(jpql)
            .sql(sql)
            .executionPlan(plan.stream()
                .map(row -> (String) row[0])
                .collect(Collectors.toList()))
            .build();
    }
    
    @Scheduled(cron = "0 0 1 * * ?") // ทุกวันตี 1
    public void reportSlowQueries() {
        // ดู slow queries จาก pg_stat_statements
        List<SlowQuery> slowQueries = entityManager
            .createNativeQuery("""
                SELECT query, calls, total_exec_time / calls as avg_ms,
                       rows / calls as avg_rows
                FROM pg_stat_statements
                WHERE calls > 100
                  AND total_exec_time / calls > 100
                ORDER BY total_exec_time / calls DESC
                LIMIT 20
                """, SlowQuery.class)
            .getResultList();
        
        if (!slowQueries.isEmpty()) {
            log.warn("พบ {} slow queries ที่ควรพิจารณา optimize", slowQueries.size());
            slowQueries.forEach(q -> 
                log.warn("Query: {}, avg: {}ms", q.getQuery(), q.getAvgMs()));
        }
    }
}
```

### Database Index Optimization

```sql
-- migration/V023__add_performance_indexes.sql

-- Orders by customer (frequent query)
CREATE INDEX CONCURRENTLY idx_orders_customer_id 
ON orders(customer_id) 
WHERE status != 'CANCELLED';  -- Partial index

-- Products full-text search
CREATE INDEX idx_products_search 
ON products USING GIN(to_tsvector('thai', name || ' ' || description));

-- Orders by date range + status
CREATE INDEX idx_orders_date_status 
ON orders(created_at DESC, status) 
INCLUDE (customer_id, total_amount);  -- Covering index

-- Analyze เพื่อ update statistics
ANALYZE orders;
ANALYZE products;
```

---

## ขั้นตอนที่ 3325: Connection Pool Sizing เพื่อลดต้นทุน DB

Database instances มี connection limit - การ over-provision connections สิ้นเปลืองทรัพยากร DB

### คำนวณ Optimal Pool Size

```
RDS db.t3.medium = 2 vCPU, 4GB RAM
Max connections = LEAST({DBInstanceClassMemory/9531392}, 5000) ≈ 420 connections

Application pods = 10
Pool per pod = 420 / 10 = 42 connections max

แต่สูตร HikariCP:
pool_per_pod = (cpu_cores × 2) + effective_spindle = (2 × 2) + 1 = 5

10 pods × 5 = 50 connections (safe margin ภายใน 420 limit)
```

```yaml
# application.yml - Optimized for cost
spring:
  datasource:
    hikari:
      minimum-idle: 2          # ลดจาก 5 → ประหยัด connections ยาม traffic ต่ำ
      maximum-pool-size: 10    # ปรับตามการคำนวณ
      idle-timeout: 300000     # ลด idle connections เร็วขึ้น (5 นาที)
      max-lifetime: 1200000    # 20 นาที
      keepalive-time: 60000    # Ping ทุก 1 นาที
```

### PgBouncer สำหรับ Connection Multiplexing

```yaml
# docker-compose.pgbouncer.yml
version: '3.8'
services:
  pgbouncer:
    image: pgbouncer/pgbouncer:latest
    environment:
      DATABASES_HOST: postgres
      DATABASES_PORT: 5432
      DATABASES_DBNAME: shophub
      PGBOUNCER_POOL_MODE: transaction  # Transaction pooling - ประหยัด connections มาก
      PGBOUNCER_MAX_CLIENT_CONN: 1000   # Applications เชื่อมได้ 1000 connections
      PGBOUNCER_DEFAULT_POOL_SIZE: 20   # แต่ DB มีแค่ 20 actual connections
      PGBOUNCER_MIN_POOL_SIZE: 5
    ports:
    - "5432:5432"
```

---

## ขั้นตอนที่ 3326: CDN สำหรับ Static Assets และ API Response Caching

CDN ลดค่า bandwidth และ load บน origin servers

### CloudFront Configuration

```java
// config/CdnConfig.java
@Configuration
public class CdnConfig {
    
    @Value("${cdn.base-url:}")
    private String cdnBaseUrl;
    
    @Bean
    public AssetUrlResolver assetUrlResolver() {
        if (StringUtils.hasText(cdnBaseUrl)) {
            return new CdnAssetUrlResolver(cdnBaseUrl);
        }
        return new LocalAssetUrlResolver();
    }
}

// resolver/CdnAssetUrlResolver.java
public class CdnAssetUrlResolver implements AssetUrlResolver {
    
    private final String cdnBaseUrl;
    
    @Override
    public String resolve(String path) {
        // https://d1234567890.cloudfront.net/images/product-1.jpg
        return cdnBaseUrl + "/assets" + path;
    }
    
    @Override
    public String resolveWithVersion(String path, String version) {
        // Cache busting ด้วย version parameter
        return cdnBaseUrl + "/assets" + path + "?v=" + version;
    }
}
```

### API Response Caching Headers

```java
// filter/CdnCacheFilter.java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CdnCacheFilter extends OncePerRequestFilter {
    
    private static final Map<String, CachePolicy> CACHE_POLICIES = Map.of(
        "/api/v1/products", new CachePolicy(300, "public"),          // 5 นาที
        "/api/v1/categories", new CachePolicy(3600, "public"),       // 1 ชั่วโมง
        "/api/v1/promotions", new CachePolicy(60, "public"),         // 1 นาที
        "/api/v1/users/", new CachePolicy(0, "private, no-store")    // ไม่ cache
    );
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        
        String path = request.getRequestURI();
        CachePolicy policy = findPolicy(path);
        
        if (policy != null && "GET".equals(request.getMethod())) {
            if (policy.maxAge() > 0) {
                response.setHeader("Cache-Control", 
                    policy.visibility() + ", max-age=" + policy.maxAge());
                response.setHeader("Vary", "Accept-Encoding, Accept-Language");
            } else {
                response.setHeader("Cache-Control", policy.visibility());
            }
        }
        
        filterChain.doFilter(request, response);
    }
    
    private CachePolicy findPolicy(String path) {
        return CACHE_POLICIES.entrySet().stream()
            .filter(e -> path.startsWith(e.getKey()))
            .map(Map.Entry::getValue)
            .findFirst()
            .orElse(null);
    }
    
    record CachePolicy(int maxAge, String visibility) {}
}
```

### Image Optimization

```java
// service/ImageOptimizationService.java
@Service
public class ImageOptimizationService {
    
    private final S3Client s3Client;
    private final String bucketName;
    
    public ImageUploadResult uploadOptimized(MultipartFile file, String productId) throws IOException {
        BufferedImage original = ImageIO.read(file.getInputStream());
        
        // สร้าง multiple sizes
        Map<String, byte[]> sizes = Map.of(
            "thumbnail", resize(original, 150, 150),
            "small", resize(original, 300, 300),
            "medium", resize(original, 600, 600),
            "large", resize(original, 1200, 1200)
        );
        
        Map<String, String> urls = new HashMap<>();
        
        for (Map.Entry<String, byte[]> entry : sizes.entrySet()) {
            String key = String.format("products/%s/%s.webp", productId, entry.getKey());
            
            // Upload WebP format (ขนาดเล็กกว่า JPEG 25-35%)
            s3Client.putObject(
                PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .contentType("image/webp")
                    .cacheControl("public, max-age=31536000") // 1 ปี
                    .build(),
                RequestBody.fromBytes(entry.getValue())
            );
            
            urls.put(entry.getKey(), cdnBaseUrl + "/" + key);
        }
        
        return ImageUploadResult.of(urls);
    }
    
    private byte[] resize(BufferedImage original, int width, int height) {
        // ใช้ Thumbnailator library
        ByteArrayOutputStream output = new ByteArrayOutputStream();
        Thumbnails.of(original)
            .size(width, height)
            .outputFormat("webp")
            .outputQuality(0.85)
            .toOutputStream(output);
        return output.toByteArray();
    }
}
```

---

## ขั้นตอนที่ 3327: Cost Monitoring กับ AWS Cost Explorer

```java
// cost/AwsCostMonitoringService.java
@Service
@ConditionalOnProperty("aws.cost-monitoring.enabled")
public class AwsCostMonitoringService {
    
    private final CostExplorerClient costExplorerClient;
    private final SlackNotificationService slackService;
    
    @Scheduled(cron = "0 0 8 * * MON") // ทุกวันจันทร์ 8 โมงเช้า
    public void sendWeeklyCostReport() {
        LocalDate endDate = LocalDate.now();
        LocalDate startDate = endDate.minusDays(7);
        
        GetCostAndUsageResponse response = costExplorerClient.getCostAndUsage(
            GetCostAndUsageRequest.builder()
                .timePeriod(DateInterval.builder()
                    .start(startDate.toString())
                    .end(endDate.toString())
                    .build())
                .granularity(Granularity.DAILY)
                .metrics(List.of("BlendedCost", "UsageQuantity"))
                .groupBy(List.of(
                    GroupDefinition.builder()
                        .type(GroupDefinitionType.DIMENSION)
                        .key("SERVICE")
                        .build()
                ))
                .filter(Expression.builder()
                    .tags(TagValues.builder()
                        .key("Project")
                        .values("ShopHub")
                        .build())
                    .build())
                .build()
        );
        
        CostReport report = processCostResponse(response);
        slackService.sendCostReport(report);
    }
    
    @Scheduled(cron = "0 0 * * * ?") // ทุกชั่วโมง
    public void checkCostAnomalies() {
        // ตรวจสอบว่า cost เพิ่มขึ้นผิดปกติ
        BigDecimal currentHourCost = getCurrentHourCost();
        BigDecimal averageHourCost = getAverageHourCost();
        
        if (currentHourCost.compareTo(averageHourCost.multiply(new BigDecimal("2"))) > 0) {
            // Cost เพิ่มขึ้น 2x จาก average → แจ้งเตือน
            slackService.sendAlert(
                String.format("⚠️ Cost anomaly! ชั่วโมงนี้: $%.2f (avg: $%.2f)",
                    currentHourCost, averageHourCost)
            );
        }
    }
    
    private CostReport processCostResponse(GetCostAndUsageResponse response) {
        // Process และ summarize cost data
        Map<String, BigDecimal> costByService = new HashMap<>();
        
        response.resultsByTime().forEach(result -> {
            result.groups().forEach(group -> {
                String service = group.keys().get(0);
                BigDecimal amount = new BigDecimal(
                    group.metrics().get("BlendedCost").amount()
                );
                costByService.merge(service, amount, BigDecimal::add);
            });
        });
        
        BigDecimal totalCost = costByService.values().stream()
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        return CostReport.builder()
            .period("Last 7 days")
            .totalCost(totalCost)
            .costByService(costByService)
            .topServices(getTopServices(costByService, 5))
            .build();
    }
}
```

### Cost Tagging Strategy

```java
// config/AwsTaggingConfig.java
@Configuration
public class AwsTaggingConfig {
    
    // Tag ทุก AWS resource เพื่อ track cost
    @Bean
    public TaggingInterceptor taggingInterceptor() {
        Map<String, String> defaultTags = Map.of(
            "Project", "ShopHub",
            "Environment", "${spring.profiles.active}",
            "Team", "backend",
            "CostCenter", "engineering"
        );
        
        return new TaggingInterceptor(defaultTags);
    }
}
```

### Savings Recommendations

```java
// cost/SavingsRecommendationService.java
@Service
public class SavingsRecommendationService {
    
    public List<SavingsRecommendation> generateRecommendations() {
        List<SavingsRecommendation> recommendations = new ArrayList<>();
        
        // 1. ตรวจสอบ idle resources
        checkIdleResources(recommendations);
        
        // 2. ตรวจสอบ oversized instances
        checkOversizedInstances(recommendations);
        
        // 3. ตรวจสอบ unattached storage
        checkUnattachedStorage(recommendations);
        
        // 4. ตรวจสอบ data transfer costs
        checkDataTransferCosts(recommendations);
        
        return recommendations.stream()
            .sorted(Comparator.comparing(SavingsRecommendation::getMonthlySavings).reversed())
            .collect(Collectors.toList());
    }
    
    private void checkIdleResources(List<SavingsRecommendation> recommendations) {
        // ดู EC2 instances ที่ CPU < 5% ตลอด 1 สัปดาห์
        cloudWatchService.getLowCpuInstances(5, Duration.ofDays(7))
            .forEach(instance -> 
                recommendations.add(SavingsRecommendation.builder()
                    .resourceId(instance.getId())
                    .type("Idle Instance")
                    .description("EC2 instance มี CPU < 5% ตลอด 7 วัน")
                    .action("Terminate หรือ downsize")
                    .monthlySavings(instance.getMonthlyCost())
                    .build())
            );
    }
}
```

---

## ขั้นตอนที่ 3328-3360: Cost Optimization Summary

### Monthly Cost Optimization Checklist

```
สัปดาห์ที่ 1:
  □ Review AWS Cost Explorer - services ไหน cost สูงที่สุด?
  □ ตรวจ heap utilization - right-size containers
  □ ดู DB query metrics - N+1 problems?
  □ ตรวจ unused Elastic IPs/NAT Gateways

สัปดาห์ที่ 2:
  □ ตรวจ RDS connection pool usage
  □ Review CloudFront cache hit rate (target > 80%)
  □ ตรวจ unused S3 objects (lifecycle policy?)
  □ Review Lambda invocation costs

สัปดาห์ที่ 3:
  □ Spot vs On-demand ratio analysis
  □ Reserved instance utilization check
  □ Data transfer costs analysis
  □ Load balancer idle check

สัปดาห์ที่ 4:
  □ Monthly cost report vs budget
  □ Savings plan recommendations
  □ Update reserved instances if needed
  □ Plan next month optimizations
```

### Cost Metrics Dashboard

```java
// actuator/CostActuatorEndpoint.java
@Component
@Endpoint(id = "cost")
public class CostActuatorEndpoint {
    
    private final CostTrackingService costTrackingService;
    
    @ReadOperation
    public Map<String, Object> cost() {
        CostReport report = costTrackingService.generateDailyCostReport();
        
        return Map.of(
            "today", Map.of(
                "estimated_usd", report.getActualCost(),
                "on_demand_equivalent_usd", report.getOnDemandEquivalent(),
                "savings_usd", report.getSavings(),
                "savings_percent", report.getSavingsPercent()
            ),
            "optimization", Map.of(
                "heap_utilization", getHeapUtilization(),
                "db_connection_utilization", getDbConnectionUtilization(),
                "cache_hit_rate", getCacheHitRate()
            )
        );
    }
    
    private double getHeapUtilization() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        return (double) memBean.getHeapMemoryUsage().getUsed() 
             / memBean.getHeapMemoryUsage().getMax() * 100;
    }
}
```

---

## สรุป

Part 93 ครอบคลุมกลยุทธ์การประหยัดต้นทุน Cloud:

1. **Right-sizing JVM** - วัด heap usage จริง แล้วปรับ container size ตาม
2. **Spot Instances** - ประหยัด 70-90% สำหรับ batch/async workloads
3. **Reserved Instances** - ประหยัด 37% สำหรับ baseline load
4. **Query Optimization** - แก้ N+1 ด้วย JOIN FETCH, เพิ่ม indexes
5. **Connection Pool** - ใช้ PgBouncer, right-size pools
6. **CDN** - ลด bandwidth costs, cache API responses
7. **Cost Monitoring** - AWS Cost Explorer + anomaly detection

ประหยัดทุก dollar ที่ประหยัดได้ = ลงทุนในการพัฒนาเพิ่มเติมได้!

---

*[← Part 92: DevOps Practices](./part-92-devops-practices.md) | [Part 94: Machine Learning Integration →](./part-94-machine-learning.md)*

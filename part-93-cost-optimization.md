# Part 93: Cloud Cost Optimization
## ขั้นตอนที่ 3321-3360

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 3-4 ชั่วโมง  
> **เป้าหมาย:** Reduce cloud costs while maintaining performance and reliability

---

## ขั้นตอนที่ 3321: Cost Categories

```
Cloud Costs for Spring Boot Apps:

Compute:
  EC2/EKS nodes  → right-size instance types
  Fargate         → pay-per-task (no idle)
  Lambda/Functions → extreme burst scaling

Database:
  RDS Multi-AZ    → expensive but required for HA
  Aurora Serverless → cost-efficient for variable load
  Read replicas   → cache to reduce read load

Network:
  Data transfer   → CDN to reduce egress costs
  NAT Gateway     → expensive per-GB

Storage:
  S3 lifecycle    → move old data to Glacier
  EBS snapshots   → automate cleanup

Optimization Levers:
  1. Right-sizing (biggest immediate savings)
  2. Reserved Instances / Savings Plans
  3. Spot for non-critical workloads
  4. Code optimization (less CPU = smaller instance)
  5. Caching (less DB queries = smaller DB)
```

---

## ขั้นตอนที่ 3322: Right-Sizing JVM in Containers

```yaml
# Kubernetes resource requests/limits
# Rule: request = what you need normally, limit = max allowed
# JVM: set -XX:MaxRAMPercentage=75 and let container limit control heap

apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              memory: "512Mi"    # normal usage
              cpu: "250m"        # 0.25 CPU normally
            limits:
              memory: "1Gi"      # max (OOM kill if exceeded)
              cpu: "1000m"       # 1 CPU max (throttled if exceeded)
          env:
            - name: JAVA_OPTS
              value: >-
                -XX:+UseContainerSupport
                -XX:MaxRAMPercentage=75.0
                -XX:+UseG1GC
```

---

## ขั้นตอนที่ 3323: Spot Instances for Batch Jobs

```yaml
# EKS node group with Spot instances for batch
# Spot can be 70-90% cheaper than On-Demand

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
managedNodeGroups:
  - name: batch-spot
    instanceTypes: ["m5.large", "m5a.large", "m4.large"]  # multiple types = more availability
    spot: true
    minSize: 0
    maxSize: 20
    labels:
      role: batch
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
```

```java
// Spring Batch job on Spot-tolerating pods
// In Job spec, add tolerations for spot nodes
// Add node affinity to prefer spot but fall back to on-demand

// Ensure jobs are checkpoint-able (Spring Batch handles this!)
// Spot can be interrupted - Spring Batch saves progress per chunk
@Bean
public Step importStep() {
    return new StepBuilder("importStep", jobRepository)
        .<Product, Product>chunk(100, transactionManager)  // commit every 100
        .reader(reader())
        .writer(writer())
        .build();
    // If spot interrupted at item 5000, restarts from item 4901 (last commit)
}
```

---

## ขั้นตอนที่ 3324: Cache to Reduce DB Costs

```java
// ทุกครั้งที่ hit cache แทน DB = ประหยัด RDS RCU
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository repository;

    // Cache product lookups (most read, rarely written)
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product findById(Long id) {
        return repository.findById(id).orElse(null);
    }

    // Cache search results (short TTL)
    @Cacheable(value = "productSearch", key = "#keyword + ':' + #page",
               condition = "#keyword.length() > 2")
    public Page<Product> search(String keyword, int page, int size) {
        return repository.searchByKeyword(keyword, PageRequest.of(page, size));
    }

    // Cache category lists (long TTL - rarely changes)
    @Cacheable(value = "categories")
    public List<Category> findAllCategories() {
        return categoryRepository.findAll();
    }
}
```

---

## ขั้นตอนที่ 3325: CDN for Static Assets

```java
// Serve static assets via CloudFront/CDN
// Add cache headers for CDN to cache responses

@Configuration
public class StaticResourceConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS)
                .cachePublic()
                .immutable());
    }
}

// API responses: add Cache-Control for CDN
@GetMapping("/api/v1/products/featured")
public ResponseEntity<List<ProductResponse>> getFeatured() {
    List<ProductResponse> products = productService.getFeatured();

    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES).cachePublic())
        .body(products);
}
```

---

## ขั้นตอนที่ 3326: S3 Lifecycle Policies

```java
// Auto-move old files to cheaper storage tiers
@Configuration
public class S3LifecycleConfig {

    @Bean
    public ApplicationRunner setupLifecyclePolicy(S3Client s3Client) {
        return args -> {
            BucketLifecycleConfiguration config = BucketLifecycleConfiguration.builder()
                .rules(
                    LifecycleRule.builder()
                        .id("move-old-logs")
                        .status(ExpirationStatus.ENABLED)
                        .filter(LifecycleRuleFilter.builder()
                            .prefix("logs/")
                            .build())
                        .transitions(
                            // Move to Infrequent Access after 30 days
                            Transition.builder()
                                .days(30)
                                .storageClass(TransitionStorageClass.STANDARD_IA)
                                .build(),
                            // Move to Glacier after 90 days
                            Transition.builder()
                                .days(90)
                                .storageClass(TransitionStorageClass.GLACIER)
                                .build()
                        )
                        .expiration(LifecycleExpiration.builder()
                            .days(365)  // Delete after 1 year
                            .build())
                        .build()
                )
                .build();

            s3Client.putBucketLifecycleConfiguration(PutBucketLifecycleConfigurationRequest.builder()
                .bucket("my-app-bucket")
                .lifecycleConfiguration(config)
                .build());
        };
    }
}
```

---

## ขั้นตอนที่ 3327-3360: Cost Monitoring Dashboard

```java
// Track application-level cost indicators
@Component
@RequiredArgsConstructor
public class CostMetricsCollector {

    private final MeterRegistry registry;
    private final ProductRepository productRepository;

    @Scheduled(fixedDelay = 60000)
    public void collectMetrics() {
        // Track DB query count (more queries = higher DB cost)
        Gauge.builder("db.queries.total",
                () -> QueryCountHolder.getGrandTotal())
            .description("Total DB queries since startup")
            .register(registry);

        // Track cache hit rate (higher = less DB cost)
        // See part-48 for cache metrics collection
    }
}

// Grafana alert: if db.queries/min > 10000, alert team to add caching
// Grafana alert: if cache.hit.rate < 80%, alert team to tune cache TTL
```

---

*[← Part 92: DevOps Practices](./part-92-devops-practices.md) | [Part 94: Machine Learning Integration →](./part-94-machine-learning.md)*

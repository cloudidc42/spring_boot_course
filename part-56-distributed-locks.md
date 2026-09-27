# Part 56: Distributed Locks
## ขั้นตอนที่ 1841-1880

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เข้าใจและนำ Distributed Locks ไปใช้งานในระบบ Microservices เพื่อแก้ปัญหา Race Conditions และรับรองความถูกต้องของข้อมูล

---

## ขั้นตอนที่ 1841-1845: ทำไมต้องใช้ Distributed Locks?

### ปัญหา Race Conditions ในระบบ Distributed

ในระบบ Microservices ที่มีหลาย instance รันพร้อมกัน เราจะเจอปัญหา Race Condition ที่ Lock ระดับ JVM ธรรมดาแก้ไม่ได้ เพราะแต่ละ process อยู่คนละเครื่อง

**ตัวอย่างปัญหา:**
- ผู้ใช้ 2 คนสั่งซื้อสินค้าชิ้นสุดท้ายพร้อมกัน
- 2 service nodes อัปเดต inventory พร้อมกัน
- Job scheduler รันงานซ้ำจาก 2 nodes พร้อมกัน

```java
// ปัญหา: ไม่มี distributed lock - race condition!
@Service
public class InventoryService {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    // ⚠️ อันตราย! ถ้ามี 2 instances รันพร้อมกัน
    public void deductStock(String productId, int quantity) {
        Inventory inventory = inventoryRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        if (inventory.getQuantity() < quantity) {
            throw new InsufficientStockException(productId);
        }
        
        // ช่วงนี้ thread อื่นอาจเข้ามาแทรก!
        inventory.setQuantity(inventory.getQuantity() - quantity);
        inventoryRepository.save(inventory);
    }
}
```

### การเพิ่ม Dependency

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Redisson - Redis-based distributed lock -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
        <version>3.23.5</version>
    </dependency>
    
    <!-- Spring Data Redis (Lettuce) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- Apache Curator (ZooKeeper) -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>5.5.0</version>
    </dependency>
    
    <!-- AOP -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 1846-1855: Redis-Based Distributed Lock ด้วย Redisson

### การ Configure Redisson

```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD:}
    
redisson:
  config: |
    singleServerConfig:
      address: "redis://localhost:6379"
      connectionPoolSize: 64
      connectionMinimumIdleSize: 10
      idleConnectionTimeout: 10000
      connectTimeout: 10000
      timeout: 3000
      retryAttempts: 3
      retryInterval: 1500
```

```java
// RedissonConfig.java
@Configuration
public class RedissonConfig {
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://localhost:6379")
            .setConnectionPoolSize(64)
            .setConnectionMinimumIdleSize(10)
            .setIdleConnectionTimeout(10000)
            .setConnectTimeout(10000)
            .setTimeout(3000)
            .setRetryAttempts(3)
            .setRetryInterval(1500);
        
        return Redisson.create(config);
    }
    
    // สำหรับ Redis Cluster
    @Bean
    @Profile("production")
    public RedissonClient redissonClusterClient() {
        Config config = new Config();
        config.useClusterServers()
            .addNodeAddress(
                "redis://redis-node1:6379",
                "redis://redis-node2:6379",
                "redis://redis-node3:6379"
            )
            .setScanInterval(2000)
            .setConnectTimeout(10000)
            .setTimeout(3000)
            .setRetryAttempts(3)
            .setRetryInterval(1500);
        
        return Redisson.create(config);
    }
}
```

### การใช้ Distributed Lock กับ Redisson

```java
// DistributedLockService.java
@Service
@Slf4j
public class DistributedLockService {
    
    private final RedissonClient redissonClient;
    
    public DistributedLockService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }
    
    /**
     * รัน action พร้อม distributed lock
     * @param lockKey - ชื่อ key ของ lock
     * @param waitTime - เวลารอสูงสุด (วินาที)
     * @param leaseTime - เวลาถือ lock สูงสุด (วินาที)
     * @param action - งานที่ต้องการรันภายใน lock
     */
    public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, 
                                  Callable<T> action) throws Exception {
        RLock lock = redissonClient.getLock(lockKey);
        
        boolean acquired = false;
        try {
            // พยายาม acquire lock
            acquired = lock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire lock ได้: " + lockKey
                );
            }
            
            log.debug("Acquired lock: {}", lockKey);
            return action.call();
            
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
                log.debug("Released lock: {}", lockKey);
            }
        }
    }
    
    /**
     * ใช้ Fair Lock สำหรับงานที่ต้องการ ordering
     */
    public <T> T executeWithFairLock(String lockKey, long waitTime, long leaseTime,
                                      Callable<T> action) throws Exception {
        RLock fairLock = redissonClient.getFairLock(lockKey);
        
        boolean acquired = false;
        try {
            acquired = fairLock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire fair lock ได้: " + lockKey
                );
            }
            
            return action.call();
            
        } finally {
            if (acquired && fairLock.isHeldByCurrentThread()) {
                fairLock.unlock();
            }
        }
    }
    
    /**
     * Multi-lock: lock หลาย resources พร้อมกัน
     */
    public <T> T executeWithMultiLock(List<String> lockKeys, long waitTime, 
                                       long leaseTime, Callable<T> action) throws Exception {
        RLock[] locks = lockKeys.stream()
            .map(key -> redissonClient.getLock(key))
            .toArray(RLock[]::new);
        
        RLock multiLock = redissonClient.getMultiLock(locks);
        
        boolean acquired = false;
        try {
            acquired = multiLock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire multi lock ได้"
                );
            }
            
            return action.call();
            
        } finally {
            if (acquired && multiLock.isHeldByCurrentThread()) {
                multiLock.unlock();
            }
        }
    }
}
```

### ปรับปรุง InventoryService ให้ใช้ Distributed Lock

```java
// InventoryService.java - แก้ไขแล้ว
@Service
@Slf4j
public class InventoryService {
    
    private static final String LOCK_PREFIX = "inventory:lock:";
    private static final long WAIT_TIME = 5L;
    private static final long LEASE_TIME = 30L;
    
    private final InventoryRepository inventoryRepository;
    private final DistributedLockService lockService;
    
    public InventoryService(InventoryRepository inventoryRepository,
                            DistributedLockService lockService) {
        this.inventoryRepository = inventoryRepository;
        this.lockService = lockService;
    }
    
    public void deductStock(String productId, int quantity) throws Exception {
        String lockKey = LOCK_PREFIX + productId;
        
        lockService.executeWithLock(lockKey, WAIT_TIME, LEASE_TIME, () -> {
            Inventory inventory = inventoryRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));
            
            if (inventory.getQuantity() < quantity) {
                throw new InsufficientStockException(productId);
            }
            
            inventory.setQuantity(inventory.getQuantity() - quantity);
            inventoryRepository.save(inventory);
            
            log.info("Deducted {} units from product {}, remaining: {}",
                quantity, productId, inventory.getQuantity());
            
            return null;
        });
    }
    
    /**
     * Transfer stock ระหว่าง warehouses - ต้องใช้ multi-lock เพื่อป้องกัน deadlock
     */
    public void transferStock(String fromWarehouse, String toWarehouse, 
                              String productId, int quantity) throws Exception {
        // เรียง lockKeys ตาม alphabetical order เพื่อป้องกัน deadlock
        List<String> lockKeys = Stream.of(
            LOCK_PREFIX + fromWarehouse + ":" + productId,
            LOCK_PREFIX + toWarehouse + ":" + productId
        ).sorted().collect(Collectors.toList());
        
        lockService.executeWithMultiLock(lockKeys, WAIT_TIME, LEASE_TIME, () -> {
            // ตัดจาก fromWarehouse
            Inventory fromInventory = inventoryRepository
                .findByWarehouseAndProductId(fromWarehouse, productId)
                .orElseThrow();
            
            if (fromInventory.getQuantity() < quantity) {
                throw new InsufficientStockException(productId);
            }
            
            fromInventory.setQuantity(fromInventory.getQuantity() - quantity);
            inventoryRepository.save(fromInventory);
            
            // เพิ่มใน toWarehouse
            Inventory toInventory = inventoryRepository
                .findByWarehouseAndProductId(toWarehouse, productId)
                .orElseGet(() -> new Inventory(toWarehouse, productId, 0));
            
            toInventory.setQuantity(toInventory.getQuantity() + quantity);
            inventoryRepository.save(toInventory);
            
            return null;
        });
    }
}
```

---

## ขั้นตอนที่ 1856-1863: @DistributedLock Custom Annotation ด้วย AOP

### สร้าง Custom Annotation

```java
// DistributedLock.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DistributedLock {
    
    /**
     * ชื่อ lock key (รองรับ SpEL expression)
     * ตัวอย่าง: "inventory:#{#productId}"
     */
    String key();
    
    /**
     * เวลารอ lock สูงสุด (default: 5 วินาที)
     */
    long waitTime() default 5L;
    
    /**
     * เวลา lease สูงสุด (default: 30 วินาที)  
     */
    long leaseTime() default 30L;
    
    /**
     * หน่วยเวลา
     */
    TimeUnit timeUnit() default TimeUnit.SECONDS;
    
    /**
     * ใช้ fair lock หรือไม่
     */
    boolean fairLock() default false;
}
```

### สร้าง AOP Aspect

```java
// DistributedLockAspect.java
@Aspect
@Component
@Slf4j
public class DistributedLockAspect {
    
    private final RedissonClient redissonClient;
    private final ExpressionParser expressionParser;
    private final ParameterNameDiscoverer parameterNameDiscoverer;
    
    public DistributedLockAspect(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
        this.expressionParser = new SpelExpressionParser();
        this.parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
    }
    
    @Around("@annotation(distributedLock)")
    public Object around(ProceedingJoinPoint joinPoint, 
                         DistributedLock distributedLock) throws Throwable {
        
        // แก้ไข SpEL key
        String lockKey = resolveLockKey(joinPoint, distributedLock.key());
        
        log.debug("Attempting to acquire distributed lock: {}", lockKey);
        
        RLock lock = distributedLock.fairLock() 
            ? redissonClient.getFairLock(lockKey)
            : redissonClient.getLock(lockKey);
        
        boolean acquired = false;
        try {
            acquired = lock.tryLock(
                distributedLock.waitTime(),
                distributedLock.leaseTime(),
                distributedLock.timeUnit()
            );
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    String.format("ไม่สามารถ acquire lock '%s' ได้ภายใน %d %s",
                        lockKey, distributedLock.waitTime(), distributedLock.timeUnit())
                );
            }
            
            log.debug("Acquired distributed lock: {}", lockKey);
            return joinPoint.proceed();
            
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
                log.debug("Released distributed lock: {}", lockKey);
            }
        }
    }
    
    private String resolveLockKey(ProceedingJoinPoint joinPoint, String keyExpression) {
        // ถ้าไม่มี SpEL expression ส่งค่ากลับตรงๆ
        if (!keyExpression.contains("#") && !keyExpression.contains("#{")) {
            return keyExpression;
        }
        
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        
        // สร้าง SpEL evaluation context
        EvaluationContext context = new StandardEvaluationContext();
        
        // ดึง parameter names
        String[] paramNames = parameterNameDiscoverer.getParameterNames(method);
        Object[] args = joinPoint.getArgs();
        
        if (paramNames != null) {
            for (int i = 0; i < paramNames.length; i++) {
                context.setVariable(paramNames[i], args[i]);
            }
        }
        
        // แก้ไข expression
        Expression expression = expressionParser.parseExpression(keyExpression);
        return expression.getValue(context, String.class);
    }
}
```

### ใช้ @DistributedLock Annotation

```java
// OrderService.java
@Service
@Slf4j
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;
    
    public OrderService(OrderRepository orderRepository,
                        InventoryRepository inventoryRepository) {
        this.orderRepository = orderRepository;
        this.inventoryRepository = inventoryRepository;
    }
    
    // Lock ตาม productId
    @DistributedLock(
        key = "order:product:#productId",
        waitTime = 10,
        leaseTime = 30
    )
    public Order createOrder(String productId, int quantity, String userId) {
        // ตรวจสอบ inventory
        Inventory inventory = inventoryRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        if (inventory.getQuantity() < quantity) {
            throw new InsufficientStockException(productId);
        }
        
        // สร้าง order
        Order order = Order.builder()
            .productId(productId)
            .quantity(quantity)
            .userId(userId)
            .status(OrderStatus.PENDING)
            .createdAt(LocalDateTime.now())
            .build();
        
        order = orderRepository.save(order);
        
        // อัปเดต inventory
        inventory.setQuantity(inventory.getQuantity() - quantity);
        inventoryRepository.save(inventory);
        
        log.info("Order created: {} for product: {}", order.getId(), productId);
        return order;
    }
    
    // Lock ตาม userId สำหรับ user-specific operations
    @DistributedLock(
        key = "user:operation:#userId",
        waitTime = 5,
        leaseTime = 15
    )
    public void processUserOperation(String userId, UserOperation operation) {
        // logic ที่ต้องการ user-level lock
        log.info("Processing operation for user: {}", userId);
    }
    
    // Fair lock สำหรับ queue processing
    @DistributedLock(
        key = "queue:processor",
        waitTime = 30,
        leaseTime = 120,
        fairLock = true
    )
    public void processNextQueueItem() {
        // process queue item ตาม order
        log.info("Processing next queue item");
    }
}
```

---

## ขั้นตอนที่ 1864-1868: ZooKeeper-Based Locks

### สร้าง ZooKeeper Lock Service

```java
// ZooKeeperLockService.java
@Service
@Slf4j
@ConditionalOnProperty(name = "lock.provider", havingValue = "zookeeper")
public class ZooKeeperLockService {
    
    private final CuratorFramework curatorClient;
    
    public ZooKeeperLockService(@Value("${zookeeper.connect-string:localhost:2181}") 
                                 String connectString) {
        this.curatorClient = CuratorFrameworkFactory.builder()
            .connectString(connectString)
            .sessionTimeoutMs(60000)
            .connectionTimeoutMs(15000)
            .retryPolicy(new ExponentialBackoffRetry(1000, 3))
            .build();
        
        this.curatorClient.start();
        
        try {
            this.curatorClient.blockUntilConnected(30, TimeUnit.SECONDS);
            log.info("Connected to ZooKeeper: {}", connectString);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("ไม่สามารถเชื่อมต่อ ZooKeeper ได้", e);
        }
    }
    
    public <T> T executeWithLock(String lockPath, long maxWaitMs, 
                                  Callable<T> action) throws Exception {
        InterProcessMutex lock = new InterProcessMutex(curatorClient, "/locks/" + lockPath);
        
        boolean acquired = false;
        try {
            acquired = lock.acquire(maxWaitMs, TimeUnit.MILLISECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire ZooKeeper lock ได้: " + lockPath
                );
            }
            
            log.debug("Acquired ZooKeeper lock: {}", lockPath);
            return action.call();
            
        } finally {
            if (acquired) {
                lock.release();
                log.debug("Released ZooKeeper lock: {}", lockPath);
            }
        }
    }
    
    /**
     * Read-Write Lock ด้วย ZooKeeper
     */
    public <T> T executeWithReadLock(String lockPath, long maxWaitMs,
                                      Callable<T> action) throws Exception {
        InterProcessReadWriteLock readWriteLock = 
            new InterProcessReadWriteLock(curatorClient, "/locks/rw/" + lockPath);
        
        InterProcessMutex readLock = readWriteLock.readLock();
        boolean acquired = false;
        
        try {
            acquired = readLock.acquire(maxWaitMs, TimeUnit.MILLISECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire read lock ได้: " + lockPath
                );
            }
            
            return action.call();
            
        } finally {
            if (acquired) {
                readLock.release();
            }
        }
    }
    
    public <T> T executeWithWriteLock(String lockPath, long maxWaitMs,
                                       Callable<T> action) throws Exception {
        InterProcessReadWriteLock readWriteLock = 
            new InterProcessReadWriteLock(curatorClient, "/locks/rw/" + lockPath);
        
        InterProcessMutex writeLock = readWriteLock.writeLock();
        boolean acquired = false;
        
        try {
            acquired = writeLock.acquire(maxWaitMs, TimeUnit.MILLISECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire write lock ได้: " + lockPath
                );
            }
            
            return action.call();
            
        } finally {
            if (acquired) {
                writeLock.release();
            }
        }
    }
    
    @PreDestroy
    public void destroy() {
        if (curatorClient != null) {
            curatorClient.close();
        }
    }
}
```

---

## ขั้นตอนที่ 1869-1873: Lock Timeouts, Lease Renewal, Deadlock Prevention

### Watchdog / Lease Renewal

```java
// LockWatchdogService.java
@Service
@Slf4j
public class LockWatchdogService {
    
    private final RedissonClient redissonClient;
    private final ScheduledExecutorService scheduler;
    private final Map<String, ScheduledFuture<?>> watchdogTasks;
    
    public LockWatchdogService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
        this.scheduler = Executors.newScheduledThreadPool(10);
        this.watchdogTasks = new ConcurrentHashMap<>();
    }
    
    /**
     * ใช้ Redisson Watchdog (built-in) - lock ไม่มีหมดอายุถ้า process ยังรันอยู่
     * เมื่อ process ตาย lock จะหมดอายุใน 30 วินาที
     */
    public <T> T executeWithWatchdog(String lockKey, Callable<T> action) throws Exception {
        // ไม่กำหนด leaseTime = -1 = ใช้ Watchdog
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // leaseTime = -1 เปิดใช้ Watchdog auto-renewal
            lock.lock();
            log.debug("Acquired lock with watchdog: {}", lockKey);
            return action.call();
            
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    /**
     * Lock พร้อม timeout callback
     */
    public <T> T executeWithTimeout(String lockKey, long timeoutSeconds, 
                                     Callable<T> action) throws Exception {
        RLock lock = redissonClient.getLock(lockKey);
        
        CompletableFuture<T> future = new CompletableFuture<>();
        
        Thread workerThread = new Thread(() -> {
            boolean acquired = false;
            try {
                acquired = lock.tryLock(5, timeoutSeconds, TimeUnit.SECONDS);
                
                if (!acquired) {
                    future.completeExceptionally(
                        new LockAcquisitionException("Lock timeout: " + lockKey)
                    );
                    return;
                }
                
                T result = action.call();
                future.complete(result);
                
            } catch (Exception e) {
                future.completeExceptionally(e);
            } finally {
                if (acquired && lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        });
        
        workerThread.start();
        
        try {
            return future.get(timeoutSeconds + 10, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            workerThread.interrupt();
            throw new LockTimeoutException("Operation timed out: " + lockKey);
        } catch (ExecutionException e) {
            throw (Exception) e.getCause();
        }
    }
}
```

### Deadlock Prevention Strategies

```java
// DeadlockPreventionService.java
@Service
@Slf4j
public class DeadlockPreventionService {
    
    private final RedissonClient redissonClient;
    
    public DeadlockPreventionService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }
    
    /**
     * Strategy 1: Lock Ordering - เรียง lock keys เสมอ
     */
    public <T> T executeWithOrderedLocks(List<String> lockKeys, Callable<T> action) 
            throws Exception {
        // เรียง alphabetically เพื่อ consistent ordering
        List<String> sortedKeys = lockKeys.stream()
            .sorted()
            .distinct()
            .collect(Collectors.toList());
        
        return acquireLocksSequentially(sortedKeys, 0, action);
    }
    
    private <T> T acquireLocksSequentially(List<String> keys, int index, 
                                            Callable<T> action) throws Exception {
        if (index >= keys.size()) {
            return action.call();
        }
        
        RLock lock = redissonClient.getLock(keys.get(index));
        boolean acquired = false;
        
        try {
            acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ acquire lock ได้: " + keys.get(index)
                );
            }
            
            return acquireLocksSequentially(keys, index + 1, action);
            
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    /**
     * Strategy 2: Try-Lock with Backoff
     */
    public <T> T executeWithBackoff(String lockKey, int maxRetries, 
                                     Callable<T> action) throws Exception {
        RLock lock = redissonClient.getLock(lockKey);
        int attempt = 0;
        
        while (attempt < maxRetries) {
            boolean acquired = false;
            try {
                // Short timeout เพื่อ fail fast
                acquired = lock.tryLock(100, 30000, TimeUnit.MILLISECONDS);
                
                if (acquired) {
                    return action.call();
                }
                
            } catch (Exception e) {
                if (acquired && lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
                throw e;
            } finally {
                if (acquired && lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
            
            attempt++;
            if (attempt < maxRetries) {
                // Exponential backoff with jitter
                long backoffMs = (long) (Math.pow(2, attempt) * 100) 
                    + (long) (Math.random() * 100);
                log.debug("Retry {} for lock {}, waiting {}ms", attempt, lockKey, backoffMs);
                Thread.sleep(backoffMs);
            }
        }
        
        throw new LockAcquisitionException(
            "ไม่สามารถ acquire lock ได้หลังจากพยายาม " + maxRetries + " ครั้ง: " + lockKey
        );
    }
}
```

---

## ขั้นตอนที่ 1874-1876: Idempotency Keys Pattern

### Idempotency Service

```java
// IdempotencyService.java
@Service
@Slf4j
public class IdempotencyService {
    
    private final RedissonClient redissonClient;
    private final ObjectMapper objectMapper;
    
    private static final String IDEMPOTENCY_KEY_PREFIX = "idempotency:";
    private static final long TTL_HOURS = 24L;
    
    public IdempotencyService(RedissonClient redissonClient, ObjectMapper objectMapper) {
        this.redissonClient = redissonClient;
        this.objectMapper = objectMapper;
    }
    
    /**
     * ประมวลผล request ด้วย idempotency
     * ถ้า key เดิมถูกใช้แล้ว ส่งคืนผลลัพธ์เดิม
     */
    public <T> T processIdempotently(String idempotencyKey, Class<T> responseType,
                                      Callable<T> action) throws Exception {
        String redisKey = IDEMPOTENCY_KEY_PREFIX + idempotencyKey;
        String lockKey = redisKey + ":lock";
        
        RBucket<String> bucket = redissonClient.getBucket(redisKey);
        
        // ตรวจสอบว่ามี cached result แล้วหรือยัง
        String cachedResult = bucket.get();
        if (cachedResult != null) {
            log.debug("Returning cached result for idempotency key: {}", idempotencyKey);
            return objectMapper.readValue(cachedResult, responseType);
        }
        
        // Acquire lock เพื่อป้องกัน duplicate processing
        RLock lock = redissonClient.getLock(lockKey);
        boolean acquired = false;
        
        try {
            acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException(
                    "ไม่สามารถ process request ได้: concurrent request detected"
                );
            }
            
            // Double-check หลัง acquire lock
            cachedResult = bucket.get();
            if (cachedResult != null) {
                return objectMapper.readValue(cachedResult, responseType);
            }
            
            // ประมวลผล
            T result = action.call();
            
            // Cache ผลลัพธ์
            String serializedResult = objectMapper.writeValueAsString(result);
            bucket.set(serializedResult, TTL_HOURS, TimeUnit.HOURS);
            
            log.info("Processed and cached result for idempotency key: {}", idempotencyKey);
            return result;
            
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### ใช้ Idempotency ใน Payment Service

```java
// PaymentService.java
@Service
@Slf4j
public class PaymentService {
    
    private final IdempotencyService idempotencyService;
    private final PaymentGateway paymentGateway;
    private final PaymentRepository paymentRepository;
    
    public PaymentService(IdempotencyService idempotencyService,
                          PaymentGateway paymentGateway,
                          PaymentRepository paymentRepository) {
        this.idempotencyService = idempotencyService;
        this.paymentGateway = paymentGateway;
        this.paymentRepository = paymentRepository;
    }
    
    public PaymentResult processPayment(PaymentRequest request) throws Exception {
        // ใช้ idempotency key จาก request header
        String idempotencyKey = request.getIdempotencyKey();
        
        return idempotencyService.processIdempotently(
            idempotencyKey,
            PaymentResult.class,
            () -> {
                // ตรวจสอบว่า payment ถูกประมวลผลแล้วหรือยัง
                Optional<Payment> existingPayment = paymentRepository
                    .findByIdempotencyKey(idempotencyKey);
                
                if (existingPayment.isPresent()) {
                    return PaymentResult.fromPayment(existingPayment.get());
                }
                
                // เรียก payment gateway
                PaymentGatewayResponse response = paymentGateway.charge(
                    request.getAmount(),
                    request.getCurrency(),
                    request.getPaymentMethod()
                );
                
                // บันทึก payment
                Payment payment = Payment.builder()
                    .idempotencyKey(idempotencyKey)
                    .amount(request.getAmount())
                    .currency(request.getCurrency())
                    .status(response.getStatus())
                    .transactionId(response.getTransactionId())
                    .processedAt(LocalDateTime.now())
                    .build();
                
                paymentRepository.save(payment);
                
                return PaymentResult.builder()
                    .transactionId(response.getTransactionId())
                    .status(response.getStatus())
                    .amount(request.getAmount())
                    .build();
            }
        );
    }
}
```

---

## ขั้นตอนที่ 1877-1880: Testing Distributed Locks

### Unit Tests

```java
// DistributedLockTest.java
@SpringBootTest
@Testcontainers
@Slf4j
class DistributedLockTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }
    
    @Autowired
    private DistributedLockService lockService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Test
    void testConcurrentLockAcquisition() throws InterruptedException {
        int threadCount = 10;
        String lockKey = "test:concurrent:lock";
        AtomicInteger counter = new AtomicInteger(0);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch finishLatch = new CountDownLatch(threadCount);
        
        List<Thread> threads = IntStream.range(0, threadCount)
            .mapToObj(i -> new Thread(() -> {
                try {
                    startLatch.await();
                    lockService.executeWithLock(lockKey, 10, 5, () -> {
                        int current = counter.get();
                        Thread.sleep(10); // simulate work
                        counter.set(current + 1);
                        return null;
                    });
                } catch (Exception e) {
                    log.error("Thread {} failed: {}", i, e.getMessage());
                } finally {
                    finishLatch.countDown();
                }
            }))
            .collect(Collectors.toList());
        
        threads.forEach(Thread::start);
        startLatch.countDown();
        finishLatch.await(30, TimeUnit.SECONDS);
        
        // ถ้า lock ทำงานถูกต้อง counter ต้องเท่ากับ threadCount
        assertThat(counter.get()).isEqualTo(threadCount);
    }
    
    @Test
    void testDeadlockPrevention() throws Exception {
        // สร้าง 2 threads ที่พยายาม acquire locks ในลำดับตรงข้าม
        // ถ้าไม่มี deadlock prevention จะ deadlock
        
        String lockA = "resource:A";
        String lockB = "resource:B";
        
        AtomicBoolean thread1Completed = new AtomicBoolean(false);
        AtomicBoolean thread2Completed = new AtomicBoolean(false);
        
        CountDownLatch latch = new CountDownLatch(2);
        
        // Thread 1: A -> B
        new Thread(() -> {
            try {
                List<String> keys = Arrays.asList(lockA, lockB);
                lockService.executeWithOrderedLocks(keys, () -> {
                    Thread.sleep(100);
                    thread1Completed.set(true);
                    return null;
                });
            } catch (Exception e) {
                log.error("Thread 1 failed", e);
            } finally {
                latch.countDown();
            }
        }).start();
        
        // Thread 2: B -> A (reverse order - แต่ service จะเรียง)
        new Thread(() -> {
            try {
                List<String> keys = Arrays.asList(lockB, lockA);
                lockService.executeWithOrderedLocks(keys, () -> {
                    Thread.sleep(100);
                    thread2Completed.set(true);
                    return null;
                });
            } catch (Exception e) {
                log.error("Thread 2 failed", e);
            } finally {
                latch.countDown();
            }
        }).start();
        
        boolean completed = latch.await(30, TimeUnit.SECONDS);
        
        assertThat(completed).isTrue();
        assertThat(thread1Completed.get()).isTrue();
        assertThat(thread2Completed.get()).isTrue();
    }
    
    @Test
    void testIdempotencyKey() throws Exception {
        String idempotencyKey = UUID.randomUUID().toString();
        AtomicInteger executionCount = new AtomicInteger(0);
        
        Callable<String> action = () -> {
            executionCount.incrementAndGet();
            return "result-" + System.currentTimeMillis();
        };
        
        // เรียกครั้งแรก
        String result1 = idempotencyService.processIdempotently(
            idempotencyKey, String.class, action
        );
        
        // เรียกครั้งสอง
        String result2 = idempotencyService.processIdempotently(
            idempotencyKey, String.class, action
        );
        
        // action ต้องถูกเรียกแค่ครั้งเดียว
        assertThat(executionCount.get()).isEqualTo(1);
        // ผลลัพธ์ต้องเหมือนกัน
        assertThat(result1).isEqualTo(result2);
    }
    
    @Test
    void testLockTimeout() {
        String lockKey = "test:timeout:lock";
        
        // Hold lock ใน background thread
        Thread holder = new Thread(() -> {
            try {
                lockService.executeWithLock(lockKey, 1, 60, () -> {
                    Thread.sleep(10000); // hold for 10 seconds
                    return null;
                });
            } catch (Exception e) {
                log.error("Holder failed", e);
            }
        });
        holder.start();
        
        Thread.sleep(100); // รอให้ holder acquire lock ก่อน
        
        // พยายาม acquire lock ที่ถูก hold อยู่
        assertThatThrownBy(() -> 
            lockService.executeWithLock(lockKey, 1, 30, () -> "should fail")
        ).isInstanceOf(LockAcquisitionException.class);
        
        holder.interrupt();
    }
}
```

### Integration Test

```java
// InventoryServiceIntegrationTest.java
@SpringBootTest
@Testcontainers
class InventoryServiceIntegrationTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @BeforeEach
    void setUp() {
        // ตั้งค่า initial inventory
        Inventory inventory = new Inventory("PRODUCT-001", 10);
        inventoryRepository.save(inventory);
    }
    
    @Test
    void testConcurrentDeduction_noOverselling() throws InterruptedException {
        int threadCount = 20; // มากกว่า stock 10 ชิ้น
        String productId = "PRODUCT-001";
        
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger failCount = new AtomicInteger(0);
        CountDownLatch latch = new CountDownLatch(threadCount);
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    inventoryService.deductStock(productId, 1);
                    successCount.incrementAndGet();
                } catch (InsufficientStockException e) {
                    failCount.incrementAndGet();
                } catch (Exception e) {
                    log.error("Unexpected error", e);
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        
        latch.await(60, TimeUnit.SECONDS);
        
        // success + fail = total threads
        assertThat(successCount.get() + failCount.get()).isEqualTo(threadCount);
        // ขาย ได้ไม่เกิน 10 ชิ้น
        assertThat(successCount.get()).isLessThanOrEqualTo(10);
        // inventory ไม่ติดลบ
        Inventory finalInventory = inventoryRepository.findById(productId).get();
        assertThat(finalInventory.getQuantity()).isGreaterThanOrEqualTo(0);
    }
}
```

---

## สรุป Part 56

ในส่วนนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Distributed Locks** - Race conditions ใน distributed systems
2. **Redisson-based locks** - การใช้ Redis เป็น lock coordinator
3. **@DistributedLock annotation** - ใช้ AOP เพื่อ declarative locking
4. **ZooKeeper-based locks** - ทางเลือกที่ใช้ ZooKeeper สำหรับ strong consistency
5. **Deadlock prevention** - lock ordering, backoff strategies
6. **Idempotency keys** - ป้องกัน duplicate processing
7. **Testing** - ทดสอบ concurrency และ lock behavior

---

*[← Part 55: Advanced Monitoring](./part-55-advanced-monitoring.md) | [Part 57: Chaos Engineering →](./part-57-chaos-engineering.md)*

# Part 22: Async & Scheduling
## ขั้นตอนที่ 576-605

> **ระดับ:** กลาง-สูง (Intermediate-Advanced)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** Async operations และ Scheduled tasks

---

## ขั้นตอนที่ 576: Async Configuration

```java
@Configuration
@EnableAsync
@Slf4j
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    @Bean(name = "taskExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 2);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async exception in method: {} with params: {}", 
                method.getName(), Arrays.toString(params), ex);
        };
    }
}
```

---

## ขั้นตอนที่ 577: @Async Methods

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {
    
    private final EmailService emailService;
    private final SmsService smsService;
    private final PushNotificationService pushService;
    
    // Fire and forget
    @Async
    public void sendWelcomeNotifications(User user) {
        log.info("Sending welcome notifications for: {}", user.getEmail());
        emailService.sendWelcomeEmail(user.getEmail(), user.getUsername());
        // If this throws, exception is handled by getAsyncUncaughtExceptionHandler
    }
    
    // Return Future - can check result
    @Async
    public CompletableFuture<Boolean> sendEmailAsync(String to, String subject, String body) {
        try {
            emailService.sendEmail(new EmailRequest(to, subject, "generic", 
                Map.of("body", body)));
            return CompletableFuture.completedFuture(true);
        } catch (Exception e) {
            log.error("Email failed: {}", e.getMessage());
            return CompletableFuture.completedFuture(false);
        }
    }
    
    // Parallel execution with multiple futures
    @Async
    public void sendAllNotifications(User user, String eventType) {
        CompletableFuture<Boolean> email = sendEmailAsync(
            user.getEmail(), eventType, "Event occurred");
        CompletableFuture<Boolean> sms = sendSmsAsync(user.getPhone(), eventType);
        CompletableFuture<Boolean> push = sendPushAsync(user.getId(), eventType);
        
        // Wait for all
        CompletableFuture.allOf(email, sms, push)
            .thenAccept(v -> log.info("All notifications sent for user: {}", user.getId()));
    }
    
    // Specific executor
    @Async("emailExecutor")
    public void sendWithEmailExecutor(String to, String message) {
        emailService.sendEmail(new EmailRequest(to, "Notification", "notification", 
            Map.of("message", message)));
    }
}
```

---

## ขั้นตอนที่ 578: CompletableFuture Patterns

```java
@Service
@RequiredArgsConstructor
public class DashboardService {
    
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final Executor taskExecutor;
    
    // Parallel data fetching
    public DashboardData getDashboardData() {
        CompletableFuture<Long> userCount = CompletableFuture
            .supplyAsync(userRepository::count, taskExecutor);
        
        CompletableFuture<Long> productCount = CompletableFuture
            .supplyAsync(productRepository::count, taskExecutor);
        
        CompletableFuture<Long> orderCount = CompletableFuture
            .supplyAsync(orderRepository::count, taskExecutor);
        
        CompletableFuture<BigDecimal> revenue = CompletableFuture
            .supplyAsync(orderRepository::calculateTotalRevenue, taskExecutor);
        
        // Wait for all and combine
        return CompletableFuture.allOf(userCount, productCount, orderCount, revenue)
            .thenApply(v -> new DashboardData(
                userCount.join(),
                productCount.join(),
                orderCount.join(),
                revenue.join()
            ))
            .exceptionally(ex -> {
                log.error("Failed to load dashboard data", ex);
                return DashboardData.empty();
            })
            .join();
    }
    
    // Chain operations
    @Async
    public CompletableFuture<OrderResponse> processOrder(CreateOrderRequest request) {
        return CompletableFuture
            .supplyAsync(() -> validateOrder(request))
            .thenApply(this::createOrder)
            .thenApply(this::calculateTotal)
            .thenApply(this::saveOrder)
            .thenApply(this::sendConfirmation)
            .exceptionally(ex -> {
                log.error("Order processing failed", ex);
                throw new BusinessException("Order processing failed: " + ex.getMessage());
            });
    }
    
    public record DashboardData(long users, long products, long orders, BigDecimal revenue) {
        static DashboardData empty() {
            return new DashboardData(0, 0, 0, BigDecimal.ZERO);
        }
    }
}
```

---

## ขั้นตอนที่ 579: Virtual Threads (Java 21)

```java
// application.yml
spring:
  threads:
    virtual:
      enabled: true  # Spring Boot 3.2+ enable virtual threads

// Config (Spring Boot 3.2+)
@Configuration
public class VirtualThreadConfig {
    
    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}

// Virtual thread executor
@Bean(name = "virtualThreadExecutor")
public Executor virtualThreadExecutor() {
    return Executors.newVirtualThreadPerTaskExecutor();
}

// Usage
@Async("virtualThreadExecutor")
public CompletableFuture<String> processWithVirtualThread() {
    // Virtual threads can block without wasting platform threads
    Thread.sleep(100);  // ไม่ block platform thread!
    return CompletableFuture.completedFuture("done");
}
```

---

## ขั้นตอนที่ 580: @Scheduled Tasks

```java
@Configuration
@EnableScheduling
public class SchedulingConfig { }

@Component
@RequiredArgsConstructor
@Slf4j
public class ScheduledTasks {
    
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final RedisTemplate<String, Object> redisTemplate;
    
    // Fixed rate - รันทุก X ms ไม่ว่า task ก่อนจะเสร็จหรือไม่
    @Scheduled(fixedRate = 60000)  // ทุก 1 นาที
    public void checkLowStockProducts() {
        List<Product> lowStock = productRepository.findLowStockProducts(10);
        if (!lowStock.isEmpty()) {
            log.warn("Low stock products: {}", lowStock.size());
            // Send alert
        }
    }
    
    // Fixed delay - รันหลังจาก task ก่อนเสร็จ + delay
    @Scheduled(fixedDelay = 30000, initialDelay = 5000)  // ทุก 30 วิ หลังเสร็จ
    public void cleanExpiredTokens() {
        // Clean up expired password reset tokens
        log.debug("Cleaning expired tokens...");
    }
    
    // Cron expression
    @Scheduled(cron = "0 0 9 * * MON-FRI")  // ทุกวันจันทร์-ศุกร์ 9:00
    public void sendDailyReport() {
        log.info("Sending daily report...");
        // Generate and send report
    }
    
    @Scheduled(cron = "0 0 0 * * *")  // ทุกเที่ยงคืน
    public void dailyCleanup() {
        log.info("Running daily cleanup...");
        redisTemplate.delete(redisTemplate.keys("cache:*"));
    }
    
    @Scheduled(cron = "0 0 0 1 * *")  // วันที่ 1 ของทุกเดือน
    public void monthlyReport() {
        log.info("Generating monthly report...");
    }
    
    // Timezone-aware cron
    @Scheduled(cron = "0 0 8 * * *", zone = "Asia/Bangkok")
    public void morningDigest() {
        log.info("Sending morning digest...");
    }
}
```

---

## ขั้นตอนที่ 581: Cron Expression Guide

```
# Format: second minute hour day-of-month month day-of-week
# ┌────────── second (0-59)
# │ ┌──────── minute (0-59)
# │ │ ┌────── hour (0-23)
# │ │ │ ┌──── day of month (1-31)
# │ │ │ │ ┌── month (1-12 or JAN-DEC)
# │ │ │ │ │ ┌ day of week (0-7 or SUN-SAT, 0 and 7 = Sunday)
# │ │ │ │ │ │
# * * * * * *

# Special characters:
# *  = any value
# ?  = any value (day-of-month and day-of-week)
# -  = range (10-12 = 10,11,12)
# ,  = list (MON,WED,FRI)
# /  = step (0/5 = 0,5,10,15...)
# L  = last (day-of-month: last day; day-of-week: last Friday = 5L)
# W  = nearest weekday (15W = nearest weekday to 15th)
# #  = nth occurrence (2#3 = 3rd Tuesday)

# Examples:
# 0 0 * * * *         = every hour at :00
# 0 */5 * * * *       = every 5 minutes
# 0 0 9-17 * * MON-FRI = 9am-5pm weekdays at :00
# 0 0 0 1 1 *         = midnight January 1st
# 0 30 10-13 ? * WED,FRI = 10:30,11:30,12:30,13:30 every Wed and Fri
```

---

## ขั้นตอนที่ 582: Dynamic Scheduling

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DynamicScheduler {
    
    private final TaskScheduler taskScheduler;
    private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();
    
    public void scheduleTask(String taskId, Runnable task, String cronExpression) {
        // Cancel existing task if any
        cancelTask(taskId);
        
        CronExpression cron = CronExpression.parse(cronExpression);
        ScheduledFuture<?> future = taskScheduler.schedule(task, new CronTrigger(cronExpression));
        
        scheduledTasks.put(taskId, future);
        log.info("Task scheduled: {} with cron: {}", taskId, cronExpression);
    }
    
    public void cancelTask(String taskId) {
        ScheduledFuture<?> future = scheduledTasks.remove(taskId);
        if (future != null) {
            future.cancel(false);
            log.info("Task cancelled: {}", taskId);
        }
    }
    
    public List<String> getScheduledTaskIds() {
        return new ArrayList<>(scheduledTasks.keySet());
    }
}
```

---

## ขั้นตอนที่ 583: Job Processing with Spring Batch (Basic)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
public class BatchConfig {
    
    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    @Bean
    public Job sendNewsletterJob() {
        return new JobBuilder("sendNewsletter", jobRepository)
            .start(sendEmailStep())
            .build();
    }
    
    @Bean
    public Step sendEmailStep() {
        return new StepBuilder("sendEmailStep", jobRepository)
            .<User, User>chunk(100, transactionManager)
            .reader(activeUserReader())
            .processor(userEmailProcessor())
            .writer(emailWriter())
            .build();
    }
    
    @Bean
    public RepositoryItemReader<User> activeUserReader() {
        RepositoryItemReader<User> reader = new RepositoryItemReader<>();
        reader.setRepository(userRepository);
        reader.setMethodName("findByActive");
        reader.setArguments(List.of(true));
        reader.setPageSize(100);
        reader.setSort(Map.of("id", Sort.Direction.ASC));
        return reader;
    }
    
    @Bean
    public ItemProcessor<User, User> userEmailProcessor() {
        return user -> {
            if (!user.isEmailVerified()) return null;  // Skip unverified
            return user;
        };
    }
    
    @Bean
    public ItemWriter<User> emailWriter() {
        return users -> {
            for (User user : users) {
                emailService.sendEmail(new EmailRequest(
                    user.getEmail(), "Newsletter", "newsletter", Map.of("user", user)
                ));
            }
        };
    }
}
```

---

## ขั้นตอนที่ 584-605: Scheduled Tasks Best Practices

```java
// ✅ Prevent duplicate execution in cluster
@Scheduled(cron = "0 0 * * * *")
public void runInCluster() {
    // Use Redis distributed lock
    String lockKey = "scheduler:hourly-report";
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "locked", Duration.ofMinutes(5));
    
    if (Boolean.TRUE.equals(acquired)) {
        try {
            doHourlyReport();
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        log.debug("Skipping - another instance is running");
    }
}

// ✅ ShedLock - distributed scheduler lock
@Scheduled(cron = "0 0 9 * * MON-FRI")
@SchedulerLock(name = "sendDailyReport", lockAtMostFor = "10m", lockAtLeastFor = "5m")
public void sendDailyReportSafe() {
    // ShedLock ensures only one node runs this
}

// ✅ Expose scheduled tasks via Actuator
@SpringBootApplication
@EnableScheduling
public class Application {
    // Add spring-boot-actuator to see /actuator/scheduledtasks
}
```

---

*[← Part 21: Redis Caching](./part-21-redis-caching.md) | [Part 23: Actuator & Monitoring →](./part-23-actuator-monitoring.md)*

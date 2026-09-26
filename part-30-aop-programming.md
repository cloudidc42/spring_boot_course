# Part 30: AOP - Aspect-Oriented Programming ขั้นสูง
## ขั้นตอนที่ 826-855

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** AOP ขั้นสูงสำหรับ Cross-Cutting Concerns

---

## ขั้นตอนที่ 826: AOP Concepts

```
AOP = Aspect-Oriented Programming
แก้ปัญหา Cross-Cutting Concerns ที่กระจายอยู่ทั่ว codebase

Cross-Cutting Concerns:
  - Logging
  - Security/Authorization
  - Transaction management
  - Performance monitoring
  - Caching
  - Error handling
  - Auditing

Key Terms:
  Aspect   = Class ที่รวม cross-cutting logic
  Advice   = Method ใน Aspect ที่จะรัน (Before, After, Around)
  Pointcut = Expression ที่ระบุว่า Advice รันที่ไหน
  JoinPoint = จุดใน code ที่ Advice รัน (method call)
  Weaving  = กระบวนการนำ Aspect มา apply กับ code
```

---

## ขั้นตอนที่ 827: Complete Audit Logging Aspect

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    String action() default "";
    String resource() default "";
    boolean logParams() default false;
    boolean logResult() default false;
}

@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class AuditAspect {
    
    private final AuditLogRepository auditLogRepository;
    private final ObjectMapper objectMapper;
    
    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        String username = getCurrentUsername();
        String ipAddress = getCurrentIpAddress();
        long startTime = System.currentTimeMillis();
        
        String resource = audited.resource().isEmpty() 
            ? pjp.getTarget().getClass().getSimpleName() 
            : audited.resource();
        
        String action = audited.action().isEmpty()
            ? pjp.getSignature().getName().toUpperCase()
            : audited.action();
        
        AuditLog log = AuditLog.builder()
            .username(username)
            .ipAddress(ipAddress)
            .resource(resource)
            .action(action)
            .method(pjp.getTarget().getClass().getSimpleName() + "." + pjp.getSignature().getName())
            .occurredAt(LocalDateTime.now())
            .build();
        
        if (audited.logParams()) {
            log.setParams(serializeParams(pjp.getArgs()));
        }
        
        try {
            Object result = pjp.proceed();
            
            log.setSuccess(true);
            log.setDurationMs(System.currentTimeMillis() - startTime);
            
            if (audited.logResult() && result != null) {
                log.setResult(objectMapper.writeValueAsString(result));
            }
            
            auditLogRepository.save(log);
            return result;
            
        } catch (Exception ex) {
            log.setSuccess(false);
            log.setErrorMessage(ex.getMessage());
            log.setDurationMs(System.currentTimeMillis() - startTime);
            
            auditLogRepository.save(log);
            throw ex;
        }
    }
    
    private String getCurrentUsername() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName)
            .orElse("anonymous");
    }
    
    private String getCurrentIpAddress() {
        return Optional.ofNullable(RequestContextHolder.getRequestAttributes())
            .filter(ServletRequestAttributes.class::isInstance)
            .map(ServletRequestAttributes.class::cast)
            .map(ServletRequestAttributes::getRequest)
            .map(req -> Optional.ofNullable(req.getHeader("X-Forwarded-For"))
                .orElse(req.getRemoteAddr()))
            .orElse("unknown");
    }
    
    private String serializeParams(Object[] args) {
        try {
            return objectMapper.writeValueAsString(
                Arrays.stream(args)
                    .map(a -> a != null ? a.toString() : "null")
                    .toArray()
            );
        } catch (JsonProcessingException e) {
            return "[serialization failed]";
        }
    }
}

// Usage
@Service
public class UserService {
    
    @Audited(action = "CREATE_USER", resource = "User")
    public UserResponse create(CreateUserRequest request) { ... }
    
    @Audited(action = "DELETE_USER", resource = "User", logParams = true)
    public void delete(Long id) { ... }
}
```

---

## ขั้นตอนที่ 828: Retry Aspect

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retryable {
    int maxAttempts() default 3;
    long delay() default 1000;
    Class<? extends Throwable>[] on() default {Exception.class};
}

@Aspect
@Component
@Slf4j
public class RetryAspect {
    
    @Around("@annotation(retryable)")
    public Object retry(ProceedingJoinPoint pjp, Retryable retryable) throws Throwable {
        int maxAttempts = retryable.maxAttempts();
        long delay = retryable.delay();
        
        Throwable lastException = null;
        
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return pjp.proceed();
            } catch (Throwable ex) {
                lastException = ex;
                
                boolean shouldRetry = Arrays.stream(retryable.on())
                    .anyMatch(c -> c.isInstance(ex));
                
                if (!shouldRetry || attempt == maxAttempts) {
                    throw ex;
                }
                
                log.warn("Attempt {}/{} failed for {}: {}. Retrying in {}ms...",
                    attempt, maxAttempts, 
                    pjp.getSignature().getName(),
                    ex.getMessage(), delay);
                
                Thread.sleep(delay * attempt);  // Exponential backoff
            }
        }
        
        throw lastException;
    }
}

// Usage
@Retryable(maxAttempts = 3, delay = 500, on = {RestClientException.class})
public UserResponse fetchFromExternalApi(Long userId) { ... }
```

---

## ขั้นตอนที่ 829: Cache Aspect (Custom)

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CustomCacheable {
    String prefix() default "";
    long ttlSeconds() default 300;
    String keyExpression() default "";
}

@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class CustomCacheAspect {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final SpelExpressionParser parser = new SpelExpressionParser();
    
    @Around("@annotation(cache)")
    public Object cache(ProceedingJoinPoint pjp, CustomCacheable cache) throws Throwable {
        String cacheKey = buildCacheKey(pjp, cache);
        
        // Check cache
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            log.debug("Cache hit: {}", cacheKey);
            return cached;
        }
        
        // Execute method
        Object result = pjp.proceed();
        
        // Store in cache
        if (result != null) {
            redisTemplate.opsForValue().set(cacheKey, result, Duration.ofSeconds(cache.ttlSeconds()));
            log.debug("Cached: {}", cacheKey);
        }
        
        return result;
    }
    
    private String buildCacheKey(ProceedingJoinPoint pjp, CustomCacheable cache) {
        String prefix = cache.prefix().isEmpty() 
            ? pjp.getSignature().getName() 
            : cache.prefix();
        
        if (!cache.keyExpression().isEmpty()) {
            // SpEL expression: "#id" → value of param named 'id'
            StandardEvaluationContext context = new StandardEvaluationContext();
            MethodSignature signature = (MethodSignature) pjp.getSignature();
            String[] paramNames = signature.getParameterNames();
            Object[] args = pjp.getArgs();
            
            for (int i = 0; i < paramNames.length; i++) {
                context.setVariable(paramNames[i], args[i]);
            }
            
            Object keyValue = parser.parseExpression(cache.keyExpression()).getValue(context);
            return prefix + ":" + keyValue;
        }
        
        return prefix + ":" + Arrays.deepHashCode(pjp.getArgs());
    }
}
```

---

## ขั้นตอนที่ 830: Rate Limit Aspect

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int requests() default 10;
    int windowSeconds() default 60;
    String key() default "";
}

@Aspect
@Component
@RequiredArgsConstructor
public class RateLimitAspect {
    
    private final RateLimiterService rateLimiterService;
    
    @Around("@annotation(rateLimit)")
    public Object rateLimit(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = buildKey(pjp, rateLimit);
        
        if (!rateLimiterService.isAllowed(key, rateLimit.requests(), 
                Duration.ofSeconds(rateLimit.windowSeconds()))) {
            throw new TooManyRequestsException("Rate limit exceeded");
        }
        
        return pjp.proceed();
    }
    
    private String buildKey(ProceedingJoinPoint pjp, RateLimit rateLimit) {
        String base = rateLimit.key().isEmpty() 
            ? pjp.getSignature().getName() 
            : rateLimit.key();
        
        // Add user identifier
        String userId = Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName)
            .orElse("anonymous");
        
        return "ratelimit:" + base + ":" + userId;
    }
}

// Usage
@PostMapping("/login")
@RateLimit(requests = 5, windowSeconds = 60)
public ResponseEntity<?> login(@RequestBody LoginRequest request) { ... }
```

---

## ขั้นตอนที่ 831: Pointcut Expressions Reference

```java
@Aspect
@Component
public class PointcutExamples {
    
    // ===== Method-level pointcuts =====
    
    // All methods in service package
    @Pointcut("execution(* com.myapp.service..*(..))")
    public void serviceLayer() {}
    
    // Specific method signature
    @Pointcut("execution(public * com.myapp.service.ProductService.find*(..))")
    public void productFindMethods() {}
    
    // Methods with @Transactional annotation
    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethods() {}
    
    // Methods returning void
    @Pointcut("execution(void com.myapp.service..*(..))")
    public void voidMethods() {}
    
    // Methods with specific parameter type
    @Pointcut("execution(* com.myapp..*(.., Long, ..))")
    public void methodsWithLongParam() {}
    
    // ===== Type-level pointcuts =====
    
    // All methods in @RestController classes
    @Pointcut("within(@org.springframework.web.bind.annotation.RestController *)")
    public void controllerLayer() {}
    
    // All methods in @Service classes
    @Pointcut("within(@org.springframework.stereotype.Service *)")
    public void allServices() {}
    
    // ===== Combined pointcuts =====
    
    @Pointcut("serviceLayer() && !transactionalMethods()")
    public void nonTransactionalService() {}
    
    // ===== Advice examples =====
    
    @Before("serviceLayer()")
    public void beforeServiceMethod(JoinPoint jp) {
        log.debug("Entering: {}.{}", 
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName());
    }
    
    @AfterReturning(pointcut = "controllerLayer()", returning = "result")
    public void afterControllerReturn(JoinPoint jp, Object result) {
        log.debug("Controller returned: {}", result);
    }
    
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void afterServiceException(JoinPoint jp, Exception ex) {
        log.error("Service exception in {}: {}", jp.getSignature().getName(), ex.getMessage());
    }
    
    @After("serviceLayer()")  // Always runs, like finally
    public void afterServiceMethod(JoinPoint jp) {
        // Cleanup
    }
}
```

---

## ขั้นตอนที่ 832-855: Complete Aspect Example

```java
// Business rule enforcement aspect
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class BusinessRuleAspect {
    
    private final UserRepository userRepository;
    
    // Prevent operations on suspended accounts
    @Before("execution(* com.myapp.service.OrderService.create(..))")
    public void checkAccountStatus(JoinPoint jp) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) return;
        
        userRepository.findByEmail(auth.getName()).ifPresent(user -> {
            if (!user.isActive()) {
                throw new BusinessException("Account is suspended");
            }
        });
    }
    
    // Log slow methods automatically
    @Around("within(com.myapp.service..*)")
    public Object logSlowMethods(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            return pjp.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 500) {
                log.warn("SLOW METHOD: {}.{} took {}ms",
                    pjp.getTarget().getClass().getSimpleName(),
                    pjp.getSignature().getName(),
                    duration);
            }
        }
    }
    
    // Soft-delete enforcement
    @Before("execution(* com.myapp.repository.*.delete*(..))")
    public void preventHardDelete(JoinPoint jp) {
        // Log warning - should use soft delete
        log.warn("Hard delete called at: {}", jp.getSignature().toShortString());
    }
}
```

---

*[← Part 29: Advanced Testing](./part-29-advanced-testing.md) | [Part 31: WebSocket →](./part-31-websocket.md)*

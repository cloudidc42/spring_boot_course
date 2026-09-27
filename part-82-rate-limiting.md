# Part 82: Rate Limiting
## ขั้นตอนที่ 2881-2920

**ระดับ:** ระดับสูง (Advanced)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้การจำกัดอัตราการเรียกใช้ API (Rate Limiting) แบบครบวงจร ตั้งแต่ algorithm พื้นฐาน ไปจนถึง Distributed Rate Limiting สำหรับระบบที่มีหลาย instance

---

## 2881-2886: Rate Limiting Algorithms

Rate Limiting เป็นเทคนิคสำคัญที่ป้องกันการใช้งาน API มากเกินไป (abuse) และรับประกัน quality of service สำหรับผู้ใช้ทุกคน

### 1. Fixed Window Algorithm

```
|<-- 1 นาที -->|<-- 1 นาที -->|
|  10 requests  |  10 requests  |
```

- นับ request ในช่วงเวลาที่กำหนด (เช่น 1 นาที)
- เมื่อครบ window ใหม่ counter จะ reset
- ข้อเสีย: burst traffic ที่ขอบ window

### 2. Sliding Window Algorithm

```
[----10 minutes window rolling----]
|--past--|--current--|
นับ requests ใน window ที่เลื่อนตามเวลาปัจจุบัน
```

- นับ request ใน sliding window (เช่น 10 นาทีที่ผ่านมา)
- แก้ปัญหา burst traffic ที่ขอบ window
- ใช้ memory มากกว่า Fixed Window

### 3. Token Bucket Algorithm

```
Token Bucket: [o][o][o][ ][ ]  capacity=5
ได้รับ token ใหม่: 1 token/วินาที
Request ใช้: 1 token/request
```

- Bucket มี tokens ที่เติมเรื่อยๆ ตามอัตราที่กำหนด
- แต่ละ request ต้องใช้ token 1 อัน
- รองรับ burst ได้ (ถ้า bucket ยังมี tokens)
- เหมาะสำหรับ API ที่อนุญาต burst แต่จำกัด sustained rate

### 4. Leaky Bucket Algorithm

```
Input  -->  [Bucket]  -->  Output ที่ rate คงที่
burst OK      |
         Overflow? DROP
```

- Request ที่เข้ามาจะ queue รอ process ที่ rate คงที่
- ถ้า queue เต็ม request จะถูก drop
- เหมาะสำหรับ smoothing burst traffic

---

## 2887-2892: Spring Cloud Gateway Rate Limiting กับ Redis

### Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### Gateway Configuration

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: api-service
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10     # tokens/วินาที
                redis-rate-limiter.burstCapacity: 20     # maximum burst
                redis-rate-limiter.requestedTokens: 1   # tokens per request
                key-resolver: "#{@userKeyResolver}"     # resolver bean

  redis:
    host: localhost
    port: 6379
```

### Key Resolvers

```java
// RateLimitConfig.java
package com.example.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.net.InetSocketAddress;

@Configuration
public class RateLimitConfig {

    // Rate limit per User (จาก JWT token)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id");
            
            if (userId != null && !userId.isBlank()) {
                return Mono.just("user:" + userId);
            }
            
            // Fallback เป็น IP ถ้าไม่มี user ID
            return Mono.just("anon:" + getClientIp(exchange));
        };
    }

    // Rate limit per API Key
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> {
            String apiKey = exchange.getRequest()
                .getHeaders()
                .getFirst("X-API-Key");
            
            if (apiKey != null && !apiKey.isBlank()) {
                return Mono.just("apikey:" + apiKey);
            }
            
            return Mono.just("anon:" + getClientIp(exchange));
        };
    }

    // Rate limit per IP Address
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            "ip:" + getClientIp(exchange)
        );
    }

    private String getClientIp(ServerWebExchange exchange) {
        // ตรวจสอบ header จาก proxy/load balancer ก่อน
        String xForwardedFor = exchange.getRequest()
            .getHeaders()
            .getFirst("X-Forwarded-For");
        
        if (xForwardedFor != null) {
            return xForwardedFor.split(",")[0].trim();
        }
        
        InetSocketAddress remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null ? remoteAddress.getHostString() : "unknown";
    }
}
```

### Custom Redis Rate Limiter

```java
// CustomRedisRateLimiter.java
package com.example.gateway.ratelimit;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.ratelimit.AbstractRateLimiter;
import org.springframework.cloud.gateway.support.ConfigurationService;
import org.springframework.data.redis.core.ReactiveRedisTemplate;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.Arrays;
import java.util.List;

@Slf4j
@Component
public class CustomRedisRateLimiter extends AbstractRateLimiter<CustomRedisRateLimiter.Config> {

    private final ReactiveRedisTemplate<String, Long> redisTemplate;
    private final RedisScript<List<Long>> script;

    public CustomRedisRateLimiter(
        ReactiveRedisTemplate<String, Long> redisTemplate,
        RedisScript<List<Long>> script,
        ConfigurationService configurationService
    ) {
        super(Config.class, "custom-rate-limiter", configurationService);
        this.redisTemplate = redisTemplate;
        this.script = script;
    }

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        Config config = getConfig().get(routeId);
        
        String key = "rate_limit:" + id;
        long now = Instant.now().getEpochSecond();
        
        return redisTemplate.execute(
            script,
            Arrays.asList(key),
            List.of(
                config.getReplenishRate(),
                config.getBurstCapacity(),
                now,
                config.getRequestedTokens()
            )
        ).reduce(1L, Long::min)
         .map(count -> {
             boolean allowed = count == 1L;
             long remaining = Math.max(0, config.getBurstCapacity() - 1);
             
             return new Response(
                 allowed,
                 getHeaders(config, remaining)
             );
         });
    }

    private Map<String, String> getHeaders(Config config, long remaining) {
        return Map.of(
            "X-RateLimit-Limit", String.valueOf(config.getBurstCapacity()),
            "X-RateLimit-Remaining", String.valueOf(remaining),
            "X-RateLimit-Reset", String.valueOf(Instant.now().plusSeconds(1).getEpochSecond())
        );
    }

    public static class Config {
        private int replenishRate = 10;
        private int burstCapacity = 20;
        private int requestedTokens = 1;
        
        // getters and setters
        public int getReplenishRate() { return replenishRate; }
        public void setReplenishRate(int replenishRate) { this.replenishRate = replenishRate; }
        public int getBurstCapacity() { return burstCapacity; }
        public void setBurstCapacity(int burstCapacity) { this.burstCapacity = burstCapacity; }
        public int getRequestedTokens() { return requestedTokens; }
        public void setRequestedTokens(int requestedTokens) { this.requestedTokens = requestedTokens; }
    }
}
```

---

## 2893-2897: Bucket4j สำหรับ In-Process Rate Limiting

Bucket4j เป็น library ที่ implement Token Bucket algorithm และรองรับทั้ง in-memory และ distributed (Redis, Hazelcast, Infinispan)

### Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.giffing.bucket4j.spring.boot.starter</groupId>
    <artifactId>bucket4j-spring-boot-starter</artifactId>
    <version>0.10.0</version>
</dependency>
<dependency>
    <groupId>io.github.bucket4j</groupId>
    <artifactId>bucket4j-redis</artifactId>
    <version>8.7.0</version>
</dependency>
```

### การตั้งค่าด้วย Configuration

```yaml
# application.yml
bucket4j:
  enabled: true
  filters:
    - cache-name: rate-limit-buckets
      url: /api/.*
      rate-limits:
        - bandwidths:
            - capacity: 100
              time: 1
              unit: minutes
              refill-speed: interval  # เติม token เป็น interval
          skip-conditions: "getHeader('X-Internal-Service') == 'true'"
          execute-predicates:
            - "Header=Authorization, .+"  # เฉพาะ requests ที่มี Authorization header
```

### การใช้งาน Bucket4j แบบ Programmatic

```java
// RateLimitService.java
package com.example.api.ratelimit;

import io.github.bucket4j.*;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

@Service
@RequiredArgsConstructor
public class RateLimitService {

    private final ProxyManager<String> proxyManager;

    // สร้าง bucket configuration ตาม plan
    private BucketConfiguration getBucketConfig(String plan) {
        return switch (plan) {
            case "FREE" -> BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(100)
                    .refillIntervally(100, Duration.ofHours(1))
                    .build())
                .build();
                
            case "PRO" -> BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(1000)
                    .refillIntervally(1000, Duration.ofHours(1))
                    .build())
                .addLimit(Bandwidth.builder()
                    .capacity(50)
                    .refillGreedy(50, Duration.ofMinutes(1))
                    .build())
                .build();
                
            case "ENTERPRISE" -> BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(10000)
                    .refillIntervally(10000, Duration.ofHours(1))
                    .build())
                .build();
                
            default -> throw new IllegalArgumentException("Unknown plan: " + plan);
        };
    }

    // ตรวจสอบ rate limit สำหรับ user
    public ConsumptionProbe checkRateLimit(String userId, String plan) {
        String key = "user:" + userId;
        Supplier<BucketConfiguration> configSupplier = () -> getBucketConfig(plan);
        
        // ดึง bucket จาก proxy manager (Redis/Hazelcast/etc)
        BucketProxy bucket = proxyManager.builder()
            .build(key, configSupplier);
        
        // ลอง consume 1 token
        return bucket.tryConsumeAndReturnRemaining(1);
    }

    // ตรวจสอบโดยไม่ consume (probe only)
    public EstimationProbe estimateRateLimit(String userId, String plan) {
        String key = "user:" + userId;
        Supplier<BucketConfiguration> configSupplier = () -> getBucketConfig(plan);
        
        BucketProxy bucket = proxyManager.builder()
            .build(key, configSupplier);
        
        return bucket.estimateAbilityToConsume(1);
    }
}
```

### Rate Limit Filter

```java
// RateLimitFilter.java
package com.example.api.filter;

import com.example.api.ratelimit.RateLimitService;
import io.github.bucket4j.ConsumptionProbe;
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Slf4j
@Component
@Order(1)
@RequiredArgsConstructor
public class RateLimitFilter implements Filter {

    private final RateLimitService rateLimitService;
    private final UserPlanService userPlanService;

    @Override
    public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String userId = httpRequest.getHeader("X-User-Id");
        
        if (userId == null) {
            chain.doFilter(request, response);
            return;
        }
        
        String plan = userPlanService.getUserPlan(userId);
        ConsumptionProbe probe = rateLimitService.checkRateLimit(userId, plan);
        
        // เพิ่ม rate limit headers
        httpResponse.setHeader("X-RateLimit-Limit", 
            String.valueOf(probe.getRemainingTokens() + probe.getNanosToWaitForRefill()));
        httpResponse.setHeader("X-RateLimit-Remaining", 
            String.valueOf(probe.getRemainingTokens()));
        
        if (probe.isConsumed()) {
            chain.doFilter(request, response);
        } else {
            // คำนวณเวลาที่ต้องรอ
            long waitSeconds = probe.getNanosToWaitForRefill() / 1_000_000_000;
            httpResponse.setHeader("Retry-After", String.valueOf(waitSeconds));
            httpResponse.setHeader("X-RateLimit-Reset", 
                String.valueOf(System.currentTimeMillis() / 1000 + waitSeconds));
            
            httpResponse.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            httpResponse.setContentType("application/json");
            httpResponse.getWriter().write(
                "{\"error\":\"Too Many Requests\",\"retryAfter\":" + waitSeconds + "}"
            );
        }
    }
}
```

---

## 2898-2903: Rate Limit Response Headers

Response headers มาตรฐานสำหรับ Rate Limiting ช่วยให้ clients รู้ว่าต้องรออีกนานแค่ไหน

### Headers มาตรฐาน

```
X-RateLimit-Limit: 100          # จำนวน requests สูงสุด
X-RateLimit-Remaining: 45       # requests ที่เหลือ
X-RateLimit-Reset: 1704067200   # Unix timestamp ที่ rate limit จะ reset
X-RateLimit-Policy: 100;w=3600  # IETF standard format
Retry-After: 30                 # วินาทีที่ต้องรอ (เมื่อถึง limit)
```

### Global Rate Limit Advice

```java
// RateLimitResponseAdvice.java
package com.example.api.advice;

import com.example.api.exception.RateLimitExceededException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.Instant;
import java.util.Map;

@RestControllerAdvice
public class RateLimitResponseAdvice {

    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<Map<String, Object>> handleRateLimitExceeded(
        RateLimitExceededException ex
    ) {
        long retryAfter = ex.getRetryAfterSeconds();
        long resetAt = Instant.now().plusSeconds(retryAfter).getEpochSecond();
        
        Map<String, Object> body = Map.of(
            "error", "Too Many Requests",
            "message", "Rate limit exceeded. Please retry after " + retryAfter + " seconds",
            "retryAfter", retryAfter,
            "resetAt", resetAt
        );
        
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .header("X-RateLimit-Remaining", "0")
            .header("X-RateLimit-Reset", String.valueOf(resetAt))
            .header("Retry-After", String.valueOf(retryAfter))
            .body(body);
    }
}
```

### Custom Exception

```java
// RateLimitExceededException.java
package com.example.api.exception;

public class RateLimitExceededException extends RuntimeException {
    
    private final long retryAfterSeconds;

    public RateLimitExceededException(long retryAfterSeconds) {
        super("Rate limit exceeded");
        this.retryAfterSeconds = retryAfterSeconds;
    }

    public long getRetryAfterSeconds() {
        return retryAfterSeconds;
    }
}
```

---

## 2904-2909: Per-User, Per-API Key, Per-IP Rate Limits

ระบบ Rate Limiting ที่ดีต้องรองรับ granularity หลายระดับ

### Multi-Level Rate Limit Service

```java
// MultiLevelRateLimitService.java
package com.example.api.ratelimit;

import io.github.bucket4j.*;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

@Slf4j
@Service
@RequiredArgsConstructor
public class MultiLevelRateLimitService {

    private final ProxyManager<String> proxyManager;

    // Rate limit per user: 1000 req/hour, max 100 req/minute
    public boolean checkUserRateLimit(String userId) {
        return tryConsume("user:" + userId, () ->
            BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(1000)
                    .refillIntervally(1000, Duration.ofHours(1))
                    .build())
                .addLimit(Bandwidth.builder()
                    .capacity(100)
                    .refillGreedy(100, Duration.ofMinutes(1))
                    .build())
                .build()
        );
    }

    // Rate limit per API key: ขึ้นอยู่กับ tier
    public boolean checkApiKeyRateLimit(String apiKey, ApiKeyTier tier) {
        return tryConsume("apikey:" + apiKey, () -> {
            int hourlyLimit = switch (tier) {
                case FREE -> 1000;
                case BASIC -> 10000;
                case PRO -> 100000;
                case ENTERPRISE -> 1000000;
            };
            
            return BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(hourlyLimit)
                    .refillIntervally(hourlyLimit, Duration.ofHours(1))
                    .build())
                .build();
        });
    }

    // Rate limit per IP: 100 req/minute สำหรับ anonymous users
    public boolean checkIpRateLimit(String ipAddress) {
        return tryConsume("ip:" + ipAddress, () ->
            BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(100)
                    .refillGreedy(100, Duration.ofMinutes(1))
                    .build())
                .build()
        );
    }

    // Rate limit per endpoint
    public boolean checkEndpointRateLimit(String userId, String endpoint) {
        String key = "endpoint:" + userId + ":" + endpoint;
        return tryConsume(key, () ->
            BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(10)
                    .refillGreedy(10, Duration.ofMinutes(1))
                    .build())
                .build()
        );
    }

    private boolean tryConsume(String key, Supplier<BucketConfiguration> configSupplier) {
        try {
            BucketProxy bucket = proxyManager.builder().build(key, configSupplier);
            return bucket.tryConsume(1);
        } catch (Exception e) {
            log.error("Error checking rate limit for key: {}", key, e);
            // Fail open: allow request ถ้า rate limit system มีปัญหา
            return true;
        }
    }

    public enum ApiKeyTier {
        FREE, BASIC, PRO, ENTERPRISE
    }
}
```

### Rate Limit Annotation

```java
// RateLimited.java
package com.example.api.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimited {
    int limit() default 100;
    int windowSeconds() default 60;
    String key() default "";  // SpEL expression
    String message() default "Rate limit exceeded";
}
```

### Rate Limit Aspect

```java
// RateLimitAspect.java
package com.example.api.aspect;

import com.example.api.annotation.RateLimited;
import com.example.api.exception.RateLimitExceededException;
import io.github.bucket4j.*;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.Expression;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.lang.reflect.Method;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Aspect
@Component
public class RateLimitAspect {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    private final ExpressionParser expressionParser = new SpelExpressionParser();

    @Around("@annotation(rateLimited)")
    public Object checkRateLimit(ProceedingJoinPoint joinPoint, RateLimited rateLimited)
        throws Throwable {
        
        String key = resolveKey(rateLimited.key(), joinPoint);
        Bucket bucket = getOrCreateBucket(key, rateLimited.limit(), rateLimited.windowSeconds());
        
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        
        if (probe.isConsumed()) {
            return joinPoint.proceed();
        } else {
            long retryAfter = probe.getNanosToWaitForRefill() / 1_000_000_000;
            throw new RateLimitExceededException(retryAfter);
        }
    }

    private String resolveKey(String keyExpression, ProceedingJoinPoint joinPoint) {
        if (keyExpression.isEmpty()) {
            // Default: ใช้ IP address
            ServletRequestAttributes attrs = 
                (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            return attrs.getRequest().getRemoteAddr();
        }
        
        // Evaluate SpEL expression
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        String[] paramNames = signature.getParameterNames();
        Object[] args = joinPoint.getArgs();
        
        EvaluationContext context = new StandardEvaluationContext();
        for (int i = 0; i < paramNames.length; i++) {
            context.setVariable(paramNames[i], args[i]);
        }
        
        Expression expression = expressionParser.parseExpression(keyExpression);
        return expression.getValue(context, String.class);
    }

    private Bucket getOrCreateBucket(String key, int limit, int windowSeconds) {
        return buckets.computeIfAbsent(key, k ->
            Bucket.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(limit)
                    .refillGreedy(limit, Duration.ofSeconds(windowSeconds))
                    .build())
                .build()
        );
    }
}
```

---

## 2910-2914: Distributed Rate Limiting

สำหรับระบบที่มีหลาย instances ต้องใช้ distributed storage (Redis) เพื่อ share rate limit state

### Redis Configuration สำหรับ Bucket4j

```java
// DistributedRateLimitConfig.java
package com.example.api.config;

import io.github.bucket4j.distributed.ExpirationAfterWriteStrategy;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import io.github.bucket4j.redis.lettuce.cas.LettuceBasedProxyManager;
import io.lettuce.core.RedisClient;
import io.lettuce.core.codec.ByteArrayCodec;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class DistributedRateLimitConfig {

    @Bean
    public ProxyManager<String> proxyManager(RedisClient redisClient) {
        return LettuceBasedProxyManager
            .builderFor(redisClient.connect(ByteArrayCodec.INSTANCE))
            .withExpirationStrategy(
                ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(
                    Duration.ofMinutes(10)
                )
            )
            .build();
    }
}
```

### Sliding Window Rate Limiter ด้วย Redis

```java
// SlidingWindowRateLimiter.java
package com.example.api.ratelimit;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.concurrent.TimeUnit;

@Slf4j
@Component
@RequiredArgsConstructor
public class SlidingWindowRateLimiter {

    private final RedisTemplate<String, String> redisTemplate;

    /**
     * Sliding Window Rate Limiting ด้วย Redis Sorted Set
     * @param key: identifier (userId, apiKey, ip)
     * @param windowSeconds: ขนาด window เป็นวินาที
     * @param maxRequests: จำนวน requests สูงสุดใน window
     * @return true ถ้า allowed
     */
    public boolean isAllowed(String key, int windowSeconds, int maxRequests) {
        String redisKey = "sliding_window:" + key;
        long now = Instant.now().toEpochMilli();
        long windowStart = now - (windowSeconds * 1000L);
        
        ZSetOperations<String, String> zSetOps = redisTemplate.opsForZSet();
        
        // ลบ entries เก่าที่อยู่นอก window
        zSetOps.removeRangeByScore(redisKey, 0, windowStart);
        
        // นับ entries ใน window ปัจจุบัน
        Long count = zSetOps.zCard(redisKey);
        
        if (count != null && count >= maxRequests) {
            log.debug("Rate limit exceeded for key: {}, count: {}/{}", key, count, maxRequests);
            return false;
        }
        
        // เพิ่ม entry ปัจจุบัน
        zSetOps.add(redisKey, String.valueOf(now), now);
        
        // ตั้ง expiration
        redisTemplate.expire(redisKey, windowSeconds * 2L, TimeUnit.SECONDS);
        
        return true;
    }

    // ดึงข้อมูล rate limit ปัจจุบัน
    public RateLimitInfo getRateLimitInfo(String key, int windowSeconds, int maxRequests) {
        String redisKey = "sliding_window:" + key;
        long now = Instant.now().toEpochMilli();
        long windowStart = now - (windowSeconds * 1000L);
        
        ZSetOperations<String, String> zSetOps = redisTemplate.opsForZSet();
        zSetOps.removeRangeByScore(redisKey, 0, windowStart);
        
        Long count = zSetOps.zCard(redisKey);
        long used = count != null ? count : 0;
        long remaining = Math.max(0, maxRequests - used);
        
        // หาเวลาที่ earliest entry จะหมดอายุ
        var oldestEntry = zSetOps.rangeWithScores(redisKey, 0, 0);
        long resetAt = now + (windowSeconds * 1000L);
        
        if (oldestEntry != null && !oldestEntry.isEmpty()) {
            var entry = oldestEntry.iterator().next();
            if (entry.getScore() != null) {
                resetAt = (long)(entry.getScore() + windowSeconds * 1000L);
            }
        }
        
        return new RateLimitInfo(maxRequests, (int) used, (int) remaining, resetAt / 1000);
    }

    public record RateLimitInfo(
        int limit,
        int used,
        int remaining,
        long resetAt
    ) {}
}
```

---

## 2915-2920: Rate Limit Bypass สำหรับ Internal Services

### Internal Service Bypass

```java
// RateLimitBypassFilter.java
package com.example.api.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import lombok.RequiredArgsConstructor;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Set;

@Component
@Order(-1)  // รันก่อน Rate Limit Filter
@RequiredArgsConstructor
public class RateLimitBypassFilter implements Filter {

    private final InternalServiceVerifier serviceVerifier;

    // เครื่องหมาย attribute ที่บอกว่า request นี้ bypass rate limit
    public static final String RATE_LIMIT_BYPASSED = "RATE_LIMIT_BYPASSED";

    @Override
    public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        if (isInternalService(httpRequest)) {
            httpRequest.setAttribute(RATE_LIMIT_BYPASSED, true);
        }
        
        chain.doFilter(request, response);
    }

    private boolean isInternalService(HttpServletRequest request) {
        // ตรวจสอบ internal service token
        String serviceToken = request.getHeader("X-Internal-Service-Token");
        if (serviceToken != null && serviceVerifier.isValidToken(serviceToken)) {
            return true;
        }
        
        // ตรวจสอบ IP ของ internal network
        String clientIp = getClientIp(request);
        return isInternalIp(clientIp);
    }

    private boolean isInternalIp(String ip) {
        // RFC 1918 private IP ranges
        return ip.startsWith("10.") ||
               ip.startsWith("172.16.") ||
               ip.startsWith("192.168.") ||
               ip.equals("127.0.0.1") ||
               ip.equals("::1");
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

### Rate Limit Configuration สำหรับ Admin Routes

```java
// AdminRouteRateLimitConfig.java
package com.example.api.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Internal endpoints ไม่ต้องผ่าน rate limiting
                .requestMatchers("/internal/**").hasRole("INTERNAL_SERVICE")
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
}
```

### Rate Limit Statistics

```java
// RateLimitStatsController.java
package com.example.api.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/internal/rate-limit")
@RequiredArgsConstructor
public class RateLimitStatsController {

    private final SlidingWindowRateLimiter rateLimiter;
    private final MultiLevelRateLimitService rateLimitService;

    @GetMapping("/stats/{userId}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Map<String, Object>> getUserStats(@PathVariable String userId) {
        var userInfo = rateLimiter.getRateLimitInfo(
            "user:" + userId,
            3600,  // 1 hour window
            1000   // 1000 requests
        );
        
        return ResponseEntity.ok(Map.of(
            "userId", userId,
            "limit", userInfo.limit(),
            "used", userInfo.used(),
            "remaining", userInfo.remaining(),
            "resetAt", userInfo.resetAt()
        ));
    }

    @DeleteMapping("/reset/{userId}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<String> resetUserRateLimit(@PathVariable String userId) {
        // ลบ rate limit data ของ user
        // (implementation depends on storage)
        return ResponseEntity.ok("Rate limit reset for user: " + userId);
    }
}
```

---

## สรุป Part 82

ในบทนี้เราได้เรียนรู้:

1. **Rate Limiting Algorithms** - Fixed Window, Sliding Window, Token Bucket, Leaky Bucket
2. **Spring Cloud Gateway** - Built-in rate limiting กับ Redis
3. **Bucket4j** - In-process และ distributed rate limiting ที่ยืดหยุ่น
4. **Response Headers** - X-RateLimit-* headers มาตรฐาน
5. **Multi-Level Rate Limiting** - Per-user, per-API-key, per-IP
6. **Internal Service Bypass** - การข้ามข้อจำกัดสำหรับ internal services

---

*[← Part 81: Scheduling Jobs](./part-81-scheduling-jobs.md) | [Part 83: Audit Trail →](./part-83-audit-trail.md)*

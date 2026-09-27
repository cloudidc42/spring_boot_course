# Part 107: โปรเจค 31-35 — Developer Tools & Infrastructure

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 31-35

---

## โปรเจค 31: URL Shortener Service

### ภาพรวม
บริการย่อ URL ที่รองรับการสร้าง Alias แบบกำหนดเอง (Custom Aliases), การติดตามจำนวนคลิก (Click Tracking), Analytics ที่ละเอียด (Referrer, Location, Browser), การกำหนดวันหมดอายุ (Expiry) และการสร้าง QR Code สำหรับลิงก์แต่ละอัน

### Dependencies (pom.xml)
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.zxing</groupId>
        <artifactId>core</artifactId>
        <version>3.5.2</version>
    </dependency>
    <dependency>
        <groupId>com.google.zxing</groupId>
        <artifactId>javase</artifactId>
        <version>3.5.2</version>
    </dependency>
    <dependency>
        <groupId>com.maxmind.geoip2</groupId>
        <artifactId>geoip2</artifactId>
        <version>4.1.0</version>
    </dependency>
</dependencies>
```

### Flyway Migration
```sql
-- V1__create_url_shortener_tables.sql
CREATE TABLE short_urls (
    id BIGSERIAL PRIMARY KEY,
    original_url TEXT NOT NULL,
    short_code VARCHAR(20) UNIQUE NOT NULL,
    custom_alias VARCHAR(50) UNIQUE,
    owner_id BIGINT,
    title VARCHAR(200),
    description TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP,
    password_hash VARCHAR(255),
    max_clicks INTEGER,
    total_clicks BIGINT DEFAULT 0,
    unique_clicks BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE url_clicks (
    id BIGSERIAL PRIMARY KEY,
    short_url_id BIGINT REFERENCES short_urls(id) ON DELETE CASCADE,
    ip_address VARCHAR(45),
    user_agent TEXT,
    referrer VARCHAR(1000),
    country_code VARCHAR(3),
    country_name VARCHAR(100),
    city VARCHAR(100),
    browser VARCHAR(50),
    os VARCHAR(50),
    device_type VARCHAR(20),
    clicked_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE url_qr_codes (
    id BIGSERIAL PRIMARY KEY,
    short_url_id BIGINT UNIQUE REFERENCES short_urls(id) ON DELETE CASCADE,
    qr_data BYTEA,
    format VARCHAR(10) DEFAULT 'PNG',
    size INTEGER DEFAULT 300,
    foreground_color VARCHAR(7) DEFAULT '#000000',
    background_color VARCHAR(7) DEFAULT '#FFFFFF',
    with_logo BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_short_urls_code ON short_urls(short_code);
CREATE INDEX idx_url_clicks_short_url ON url_clicks(short_url_id, clicked_at DESC);
```

### Entity Classes
```java
// ShortUrl.java
@Entity
@Table(name = "short_urls")
@Data
@NoArgsConstructor
public class ShortUrl {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "original_url", columnDefinition = "TEXT", nullable = false)
    private String originalUrl;

    @Column(name = "short_code", unique = true, nullable = false)
    private String shortCode;

    @Column(name = "custom_alias", unique = true)
    private String customAlias;

    @Column(name = "owner_id")
    private Long ownerId;

    private String title;

    @Column(name = "is_active")
    private Boolean isActive = true;

    @Column(name = "expires_at")
    private LocalDateTime expiresAt;

    @Column(name = "password_hash")
    private String passwordHash;

    @Column(name = "max_clicks")
    private Integer maxClicks;

    @Column(name = "total_clicks")
    private Long totalClicks = 0L;

    @Column(name = "unique_clicks")
    private Long uniqueClicks = 0L;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    public String getEffectiveCode() {
        return customAlias != null ? customAlias : shortCode;
    }

    public boolean isExpired() {
        return expiresAt != null && LocalDateTime.now().isAfter(expiresAt);
    }

    public boolean hasReachedMaxClicks() {
        return maxClicks != null && totalClicks >= maxClicks;
    }
}

// UrlClick.java
@Entity
@Table(name = "url_clicks")
@Data
@NoArgsConstructor
public class UrlClick {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "short_url_id", nullable = false)
    private ShortUrl shortUrl;

    @Column(name = "ip_address")
    private String ipAddress;

    @Column(name = "user_agent", columnDefinition = "TEXT")
    private String userAgent;

    private String referrer;

    @Column(name = "country_code")
    private String countryCode;

    @Column(name = "country_name")
    private String countryName;

    private String city;
    private String browser;
    private String os;

    @Column(name = "device_type")
    private String deviceType;

    @Column(name = "clicked_at")
    private LocalDateTime clickedAt = LocalDateTime.now();
}
```

### Service
```java
// UrlShortenerService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class UrlShortenerService {

    private final ShortUrlRepository shortUrlRepository;
    private final UrlClickRepository clickRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final GeoIpService geoIpService;
    private final UserAgentParser userAgentParser;

    private static final String REDIRECT_CACHE_PREFIX = "url:redirect:";
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    private static final String BASE_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

    public ShortUrl createShortUrl(CreateUrlRequest request, Long ownerId) {
        if (request.getCustomAlias() != null) {
            if (shortUrlRepository.existsByCustomAlias(request.getCustomAlias())) {
                throw new ConflictException("Custom alias already taken: " + request.getCustomAlias());
            }
            validateAlias(request.getCustomAlias());
        }

        ShortUrl shortUrl = new ShortUrl();
        shortUrl.setOriginalUrl(request.getOriginalUrl());
        shortUrl.setShortCode(generateUniqueCode());
        shortUrl.setCustomAlias(request.getCustomAlias());
        shortUrl.setOwnerId(ownerId);
        shortUrl.setTitle(request.getTitle());
        shortUrl.setExpiresAt(request.getExpiresAt());
        shortUrl.setMaxClicks(request.getMaxClicks());

        if (request.getPassword() != null) {
            shortUrl.setPasswordHash(passwordEncoder.encode(request.getPassword()));
        }

        ShortUrl saved = shortUrlRepository.save(shortUrl);
        cacheRedirect(saved.getEffectiveCode(), saved.getOriginalUrl());

        return saved;
    }

    public String resolveUrl(String code, HttpServletRequest httpRequest) {
        // Check cache first
        String cachedUrl = redisTemplate.opsForValue().get(REDIRECT_CACHE_PREFIX + code);
        if (cachedUrl != null) {
            // Record click asynchronously
            recordClickAsync(code, httpRequest);
            return cachedUrl;
        }

        ShortUrl shortUrl = shortUrlRepository.findByShortCodeOrCustomAlias(code, code)
                .orElseThrow(() -> new ResourceNotFoundException("Short URL not found: " + code));

        if (!shortUrl.getIsActive()) {
            throw new BusinessException("This link is no longer active");
        }
        if (shortUrl.isExpired()) {
            throw new BusinessException("This link has expired");
        }
        if (shortUrl.hasReachedMaxClicks()) {
            throw new BusinessException("This link has reached its maximum click limit");
        }

        cacheRedirect(code, shortUrl.getOriginalUrl());
        recordClickAsync(code, httpRequest);

        return shortUrl.getOriginalUrl();
    }

    @Async
    public void recordClickAsync(String code, HttpServletRequest request) {
        try {
            ShortUrl shortUrl = shortUrlRepository.findByShortCodeOrCustomAlias(code, code).orElse(null);
            if (shortUrl == null) return;

            String ipAddress = extractIpAddress(request);
            String userAgent = request.getHeader("User-Agent");
            String referrer = request.getHeader("Referer");

            UrlClick click = new UrlClick();
            click.setShortUrl(shortUrl);
            click.setIpAddress(ipAddress);
            click.setUserAgent(userAgent);
            click.setReferrer(referrer);

            // Parse user agent
            UserAgentInfo uaInfo = userAgentParser.parse(userAgent);
            click.setBrowser(uaInfo.getBrowser());
            click.setOs(uaInfo.getOs());
            click.setDeviceType(uaInfo.getDeviceType());

            // GeoIP lookup
            GeoLocation location = geoIpService.lookup(ipAddress);
            if (location != null) {
                click.setCountryCode(location.getCountryCode());
                click.setCountryName(location.getCountryName());
                click.setCity(location.getCity());
            }

            clickRepository.save(click);
            shortUrlRepository.incrementClicks(shortUrl.getId());

        } catch (Exception e) {
            log.error("Failed to record click for code {}: {}", code, e.getMessage());
        }
    }

    public UrlAnalytics getAnalytics(Long shortUrlId, LocalDateTime from, LocalDateTime to) {
        List<UrlClick> clicks = clickRepository.findByShortUrlIdAndDateRange(shortUrlId, from, to);

        Map<String, Long> clicksByCountry = clicks.stream()
                .filter(c -> c.getCountryName() != null)
                .collect(Collectors.groupingBy(UrlClick::getCountryName, Collectors.counting()));

        Map<String, Long> clicksByBrowser = clicks.stream()
                .filter(c -> c.getBrowser() != null)
                .collect(Collectors.groupingBy(UrlClick::getBrowser, Collectors.counting()));

        Map<String, Long> clicksByDevice = clicks.stream()
                .filter(c -> c.getDeviceType() != null)
                .collect(Collectors.groupingBy(UrlClick::getDeviceType, Collectors.counting()));

        Map<LocalDate, Long> clicksByDay = clicks.stream()
                .collect(Collectors.groupingBy(
                        c -> c.getClickedAt().toLocalDate(), Collectors.counting()));

        return new UrlAnalytics(clicks.size(), clicksByCountry, clicksByBrowser, clicksByDevice, clicksByDay);
    }

    private String generateUniqueCode() {
        String code;
        do {
            code = generateCode(7);
        } while (shortUrlRepository.existsByShortCode(code));
        return code;
    }

    private String generateCode(int length) {
        SecureRandom random = new SecureRandom();
        StringBuilder sb = new StringBuilder(length);
        for (int i = 0; i < length; i++) {
            sb.append(BASE_CHARS.charAt(random.nextInt(BASE_CHARS.length())));
        }
        return sb.toString();
    }

    private void cacheRedirect(String code, String url) {
        redisTemplate.opsForValue().set(REDIRECT_CACHE_PREFIX + code, url, CACHE_TTL);
    }
}

// QrCodeService.java
@Service
public class QrCodeService {

    public byte[] generateQrCode(String url, int size, String foregroundColor, String backgroundColor) throws Exception {
        Map<EncodeHintType, Object> hints = new HashMap<>();
        hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
        hints.put(EncodeHintType.MARGIN, 2);

        QRCodeWriter writer = new QRCodeWriter();
        BitMatrix matrix = writer.encode(url, BarcodeFormat.QR_CODE, size, size, hints);

        BufferedImage image = new BufferedImage(size, size, BufferedImage.TYPE_INT_RGB);
        int fg = Color.decode(foregroundColor).getRGB();
        int bg = Color.decode(backgroundColor).getRGB();

        for (int x = 0; x < size; x++) {
            for (int y = 0; y < size; y++) {
                image.setRGB(x, y, matrix.get(x, y) ? fg : bg);
            }
        }

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ImageIO.write(image, "PNG", baos);
        return baos.toByteArray();
    }
}
```

### Controller
```java
// UrlShortenerController.java
@RestController
@RequestMapping("/api/urls")
@RequiredArgsConstructor
public class UrlShortenerController {

    private final UrlShortenerService urlService;
    private final QrCodeService qrCodeService;

    @PostMapping
    public ResponseEntity<ShortUrlDTO> createShortUrl(@RequestBody @Valid CreateUrlRequest request,
                                                        Authentication auth) {
        Long ownerId = auth != null ? getCurrentUserId(auth) : null;
        ShortUrl shortUrl = urlService.createShortUrl(request, ownerId);
        return ResponseEntity.status(HttpStatus.CREATED).body(ShortUrlDTO.from(shortUrl));
    }

    @GetMapping("/{code}")
    public ResponseEntity<ShortUrlDTO> getUrlInfo(@PathVariable String code) {
        return ResponseEntity.ok(ShortUrlDTO.from(urlService.getByCode(code)));
    }

    @GetMapping("/{id}/analytics")
    public ResponseEntity<UrlAnalytics> getAnalytics(
            @PathVariable Long id,
            @RequestParam(defaultValue = "#{T(java.time.LocalDateTime).now().minusDays(30)}") LocalDateTime from,
            @RequestParam(defaultValue = "#{T(java.time.LocalDateTime).now()}") LocalDateTime to) {
        return ResponseEntity.ok(urlService.getAnalytics(id, from, to));
    }

    @GetMapping("/{code}/qr")
    public ResponseEntity<byte[]> getQrCode(
            @PathVariable String code,
            @RequestParam(defaultValue = "300") int size,
            @RequestParam(defaultValue = "#000000") String fg,
            @RequestParam(defaultValue = "#FFFFFF") String bg) throws Exception {
        ShortUrl shortUrl = urlService.getByCode(code);
        byte[] qrBytes = qrCodeService.generateQrCode(shortUrl.getOriginalUrl(), size, fg, bg);
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_PNG)
                .body(qrBytes);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUrl(@PathVariable Long id, Authentication auth) {
        urlService.deleteUrl(id, getCurrentUserId(auth));
        return ResponseEntity.noContent().build();
    }
}

// RedirectController.java
@Controller
@RequiredArgsConstructor
public class RedirectController {

    private final UrlShortenerService urlService;

    @GetMapping("/{code}")
    public ResponseEntity<Void> redirect(@PathVariable String code, HttpServletRequest request) {
        String originalUrl = urlService.resolveUrl(code, request);
        return ResponseEntity.status(HttpStatus.FOUND)
                .location(URI.create(originalUrl))
                .build();
    }
}
```

---

## โปรเจค 32: Feature Flag Service

### ภาพรวม
ระบบ Feature Flags ที่รองรับการเปิด/ปิดฟีเจอร์ตาม Environment, การ Rollout แบบ Percentage, การกำหนดเป้าหมายผู้ใช้ (User Targeting), การแบ่งกลุ่ม A/B Testing, Audit Log และ SDK Endpoint สำหรับแอปต่างๆ ในการ Query

### Flyway Migration
```sql
-- V1__create_feature_flag_tables.sql
CREATE TABLE feature_flags (
    id BIGSERIAL PRIMARY KEY,
    key VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    enabled BOOLEAN DEFAULT FALSE,
    created_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE flag_environments (
    id BIGSERIAL PRIMARY KEY,
    flag_id BIGINT REFERENCES feature_flags(id) ON DELETE CASCADE,
    environment VARCHAR(50) NOT NULL,
    enabled BOOLEAN DEFAULT FALSE,
    rollout_percentage INTEGER DEFAULT 0 CHECK (rollout_percentage BETWEEN 0 AND 100),
    targeting_rules JSONB,
    ab_group_config JSONB,
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(flag_id, environment)
);

CREATE TABLE flag_audit_log (
    id BIGSERIAL PRIMARY KEY,
    flag_id BIGINT REFERENCES feature_flags(id),
    action VARCHAR(50) NOT NULL,
    environment VARCHAR(50),
    old_value JSONB,
    new_value JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE flag_sdk_tokens (
    id BIGSERIAL PRIMARY KEY,
    token VARCHAR(100) UNIQUE NOT NULL,
    environment VARCHAR(50) NOT NULL,
    description VARCHAR(200),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Entity & Service
```java
// FeatureFlag.java
@Entity
@Table(name = "feature_flags")
@Data
@NoArgsConstructor
public class FeatureFlag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String key;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    private Boolean enabled = false;

    @Column(name = "created_by")
    private String createdBy;

    @OneToMany(mappedBy = "flag", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<FlagEnvironment> environments = new ArrayList<>();
}

// FlagEnvironment.java
@Entity
@Table(name = "flag_environments")
@Data
@NoArgsConstructor
public class FlagEnvironment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "flag_id")
    private FeatureFlag flag;

    @Column(nullable = false)
    private String environment;

    private Boolean enabled = false;

    @Column(name = "rollout_percentage")
    private Integer rolloutPercentage = 0;

    @Type(JsonType.class)
    @Column(name = "targeting_rules", columnDefinition = "jsonb")
    private List<TargetingRule> targetingRules;

    @Type(JsonType.class)
    @Column(name = "ab_group_config", columnDefinition = "jsonb")
    private AbGroupConfig abGroupConfig;
}

// FeatureFlagService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class FeatureFlagService {

    private final FeatureFlagRepository flagRepository;
    private final FlagEnvironmentRepository envRepository;
    private final FlagAuditLogRepository auditRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String FLAG_CACHE_PREFIX = "flags:";

    @Cacheable(value = "flags", key = "#key + ':' + #environment")
    public boolean isEnabled(String key, String environment, EvaluationContext context) {
        FlagEnvironment flagEnv = envRepository.findByFlagKeyAndEnvironment(key, environment)
                .orElse(null);

        if (flagEnv == null || !flagEnv.getEnabled()) {
            return false;
        }

        // Check targeting rules first
        if (flagEnv.getTargetingRules() != null && !flagEnv.getTargetingRules().isEmpty()) {
            for (TargetingRule rule : flagEnv.getTargetingRules()) {
                if (matchesRule(rule, context)) {
                    return rule.getServeEnabled();
                }
            }
        }

        // Check rollout percentage
        if (flagEnv.getRolloutPercentage() > 0 && flagEnv.getRolloutPercentage() < 100) {
            return isInRollout(context.getUserId(), key, flagEnv.getRolloutPercentage());
        }

        return flagEnv.getRolloutPercentage() == 100;
    }

    public Map<String, Boolean> evaluateAllFlags(String environment, EvaluationContext context) {
        List<FlagEnvironment> envFlags = envRepository.findByEnvironment(environment);
        Map<String, Boolean> results = new HashMap<>();

        for (FlagEnvironment flagEnv : envFlags) {
            String key = flagEnv.getFlag().getKey();
            results.put(key, isEnabled(key, environment, context));
        }

        return results;
    }

    public FeatureFlag createFlag(CreateFlagRequest request, String createdBy) {
        if (flagRepository.existsByKey(request.getKey())) {
            throw new ConflictException("Flag key already exists: " + request.getKey());
        }

        FeatureFlag flag = new FeatureFlag();
        flag.setKey(request.getKey());
        flag.setName(request.getName());
        flag.setDescription(request.getDescription());
        flag.setCreatedBy(createdBy);

        FeatureFlag saved = flagRepository.save(flag);

        // Create environment configs for standard environments
        List.of("development", "staging", "production").forEach(env -> {
            FlagEnvironment flagEnv = new FlagEnvironment();
            flagEnv.setFlag(saved);
            flagEnv.setEnvironment(env);
            flagEnv.setEnabled(false);
            envRepository.save(flagEnv);
        });

        return saved;
    }

    public FlagEnvironment updateFlagForEnvironment(String key, String environment,
                                                     UpdateFlagEnvRequest request, String changedBy) {
        FlagEnvironment flagEnv = envRepository.findByFlagKeyAndEnvironment(key, environment)
                .orElseThrow(() -> new ResourceNotFoundException("Flag environment not found"));

        FlagEnvironment oldValue = SerializationUtils.clone(flagEnv);

        flagEnv.setEnabled(request.getEnabled());
        if (request.getRolloutPercentage() != null) {
            flagEnv.setRolloutPercentage(request.getRolloutPercentage());
        }
        if (request.getTargetingRules() != null) {
            flagEnv.setTargetingRules(request.getTargetingRules());
        }

        FlagEnvironment saved = envRepository.save(flagEnv);

        // Audit log
        logAudit(flagEnv.getFlag().getId(), "UPDATE_ENVIRONMENT", environment,
                oldValue, saved, changedBy);

        // Invalidate cache
        evictFlagCache(key, environment);

        return saved;
    }

    private boolean isInRollout(String userId, String flagKey, int percentage) {
        if (userId == null) return false;
        String hash = flagKey + ":" + userId;
        int hashValue = Math.abs(hash.hashCode()) % 100;
        return hashValue < percentage;
    }

    private boolean matchesRule(TargetingRule rule, EvaluationContext context) {
        if ("USER_ID".equals(rule.getAttribute())) {
            return rule.getValues().contains(context.getUserId());
        }
        if ("USER_ATTRIBUTE".equals(rule.getAttribute())) {
            String attrValue = context.getAttributes().get(rule.getAttributeKey());
            return rule.getValues().contains(attrValue);
        }
        return false;
    }

    private void logAudit(Long flagId, String action, String environment,
                           Object oldValue, Object newValue, String changedBy) {
        FlagAuditLog log = new FlagAuditLog();
        log.setFlagId(flagId);
        log.setAction(action);
        log.setEnvironment(environment);
        log.setChangedBy(changedBy);
        auditRepository.save(log);
    }
}
```

### Controller
```java
// FeatureFlagController.java
@RestController
@RequestMapping("/api/flags")
@RequiredArgsConstructor
public class FeatureFlagController {

    private final FeatureFlagService flagService;

    @PostMapping
    public ResponseEntity<FlagDTO> createFlag(@RequestBody @Valid CreateFlagRequest request,
                                               Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(FlagDTO.from(flagService.createFlag(request, auth.getName())));
    }

    @GetMapping
    public ResponseEntity<List<FlagDTO>> listFlags() {
        return ResponseEntity.ok(flagService.listAllFlags());
    }

    @PutMapping("/{key}/environments/{env}")
    public ResponseEntity<FlagEnvDTO> updateFlagEnvironment(
            @PathVariable String key,
            @PathVariable String env,
            @RequestBody @Valid UpdateFlagEnvRequest request,
            Authentication auth) {
        return ResponseEntity.ok(FlagEnvDTO.from(
                flagService.updateFlagForEnvironment(key, env, request, auth.getName())));
    }

    @GetMapping("/{key}/audit-log")
    public ResponseEntity<List<AuditLogDTO>> getAuditLog(@PathVariable String key) {
        return ResponseEntity.ok(flagService.getAuditLog(key));
    }
}

// FlagSdkController.java - For apps to query flags
@RestController
@RequestMapping("/sdk/flags")
@RequiredArgsConstructor
public class FlagSdkController {

    private final FeatureFlagService flagService;
    private final SdkTokenValidator tokenValidator;

    @GetMapping
    public ResponseEntity<Map<String, Boolean>> getAllFlags(
            @RequestHeader("X-SDK-Token") String sdkToken,
            @RequestParam(required = false) String userId,
            @RequestParam(required = false) Map<String, String> attributes) {
        String environment = tokenValidator.validateAndGetEnvironment(sdkToken);
        EvaluationContext context = new EvaluationContext(userId, attributes);
        return ResponseEntity.ok(flagService.evaluateAllFlags(environment, context));
    }

    @GetMapping("/{key}")
    public ResponseEntity<Map<String, Boolean>> getFlag(
            @PathVariable String key,
            @RequestHeader("X-SDK-Token") String sdkToken,
            @RequestParam(required = false) String userId,
            @RequestParam(required = false) Map<String, String> attributes) {
        String environment = tokenValidator.validateAndGetEnvironment(sdkToken);
        EvaluationContext context = new EvaluationContext(userId, attributes);
        boolean enabled = flagService.isEnabled(key, environment, context);
        return ResponseEntity.ok(Map.of("enabled", enabled));
    }
}
```

---

## โปรเจค 33: Webhook Platform

### ภาพรวม
แพลตฟอร์ม Webhook ที่รองรับการลงทะเบียน Webhook, การ Subscribe Event, การส่งพร้อม Retry อัตโนมัติ, บันทึกการส่ง (Delivery Logs), การยืนยันด้วย HMAC Signature และการส่งซ้ำ (Replay) สำหรับการส่งที่ล้มเหลว

### Flyway Migration
```sql
-- V1__create_webhook_tables.sql
CREATE TABLE webhook_endpoints (
    id BIGSERIAL PRIMARY KEY,
    owner_id BIGINT NOT NULL,
    url VARCHAR(1000) NOT NULL,
    description VARCHAR(500),
    secret VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    timeout_seconds INTEGER DEFAULT 30,
    max_retries INTEGER DEFAULT 3,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE webhook_subscriptions (
    id BIGSERIAL PRIMARY KEY,
    endpoint_id BIGINT REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(endpoint_id, event_type)
);

CREATE TABLE webhook_deliveries (
    id BIGSERIAL PRIMARY KEY,
    endpoint_id BIGINT REFERENCES webhook_endpoints(id),
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    attempt_count INTEGER DEFAULT 0,
    next_retry_at TIMESTAMP,
    last_response_code INTEGER,
    last_response_body TEXT,
    last_error TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    delivered_at TIMESTAMP
);

CREATE TABLE webhook_delivery_attempts (
    id BIGSERIAL PRIMARY KEY,
    delivery_id BIGINT REFERENCES webhook_deliveries(id) ON DELETE CASCADE,
    attempt_number INTEGER NOT NULL,
    response_code INTEGER,
    response_body TEXT,
    duration_ms INTEGER,
    error_message TEXT,
    attempted_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhook_deliveries_status ON webhook_deliveries(status, next_retry_at);
```

### Service
```java
// WebhookService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class WebhookService {

    private final WebhookEndpointRepository endpointRepository;
    private final WebhookDeliveryRepository deliveryRepository;
    private final WebhookAttemptRepository attemptRepository;
    private final WebhookSubscriptionRepository subscriptionRepository;
    private final RestTemplate restTemplate;

    private static final long[] RETRY_DELAYS_SECONDS = {30, 300, 1800, 7200, 86400};

    @Transactional
    public void dispatchEvent(String eventType, Object payload) {
        List<WebhookEndpoint> endpoints = endpointRepository
                .findActiveEndpointsByEventType(eventType);

        for (WebhookEndpoint endpoint : endpoints) {
            WebhookDelivery delivery = new WebhookDelivery();
            delivery.setEndpoint(endpoint);
            delivery.setEventType(eventType);
            delivery.setPayload(convertToMap(payload));
            delivery.setStatus(WebhookDelivery.Status.PENDING);
            WebhookDelivery saved = deliveryRepository.save(delivery);

            // Send asynchronously
            CompletableFuture.runAsync(() -> attemptDelivery(saved.getId()));
        }
    }

    @Async
    public void attemptDelivery(Long deliveryId) {
        WebhookDelivery delivery = deliveryRepository.findById(deliveryId)
                .orElseThrow(() -> new ResourceNotFoundException("Delivery not found"));

        WebhookEndpoint endpoint = delivery.getEndpoint();
        String payload = buildPayload(delivery);
        String signature = generateSignature(payload, endpoint.getSecret());

        long startTime = System.currentTimeMillis();
        WebhookAttempt attempt = new WebhookAttempt();
        attempt.setDelivery(delivery);
        attempt.setAttemptNumber(delivery.getAttemptCount() + 1);

        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.set("X-Webhook-Signature", "sha256=" + signature);
            headers.set("X-Webhook-Event", delivery.getEventType());
            headers.set("X-Webhook-Delivery-Id", delivery.getId().toString());
            headers.set("X-Webhook-Timestamp", String.valueOf(System.currentTimeMillis()));

            HttpEntity<String> entity = new HttpEntity<>(payload, headers);
            ResponseEntity<String> response = restTemplate.exchange(
                    endpoint.getUrl(), HttpMethod.POST, entity, String.class);

            int statusCode = response.getStatusCode().value();
            attempt.setResponseCode(statusCode);
            attempt.setResponseBody(response.getBody());

            if (statusCode >= 200 && statusCode < 300) {
                delivery.setStatus(WebhookDelivery.Status.DELIVERED);
                delivery.setDeliveredAt(LocalDateTime.now());
                log.info("Webhook delivered successfully to {} for event {}",
                        endpoint.getUrl(), delivery.getEventType());
            } else {
                handleDeliveryFailure(delivery, "HTTP " + statusCode, endpoint);
            }

        } catch (Exception e) {
            log.error("Webhook delivery failed: {}", e.getMessage());
            attempt.setErrorMessage(e.getMessage());
            handleDeliveryFailure(delivery, e.getMessage(), endpoint);
        } finally {
            attempt.setDurationMs((int)(System.currentTimeMillis() - startTime));
            attemptRepository.save(attempt);
        }

        delivery.setAttemptCount(delivery.getAttemptCount() + 1);
        delivery.setLastResponseCode(attempt.getResponseCode());
        deliveryRepository.save(delivery);
    }

    private void handleDeliveryFailure(WebhookDelivery delivery, String error, WebhookEndpoint endpoint) {
        delivery.setLastError(error);
        int attempts = delivery.getAttemptCount();

        if (attempts < endpoint.getMaxRetries() && attempts < RETRY_DELAYS_SECONDS.length) {
            delivery.setStatus(WebhookDelivery.Status.RETRYING);
            long delaySeconds = RETRY_DELAYS_SECONDS[attempts];
            delivery.setNextRetryAt(LocalDateTime.now().plusSeconds(delaySeconds));
            log.info("Scheduling retry {} for delivery {} in {}s", attempts + 1, delivery.getId(), delaySeconds);
        } else {
            delivery.setStatus(WebhookDelivery.Status.FAILED);
            log.warn("Webhook delivery {} permanently failed after {} attempts", delivery.getId(), attempts);
        }
    }

    public WebhookDelivery replayDelivery(Long deliveryId) {
        WebhookDelivery original = deliveryRepository.findById(deliveryId)
                .orElseThrow(() -> new ResourceNotFoundException("Delivery not found"));

        WebhookDelivery replay = new WebhookDelivery();
        replay.setEndpoint(original.getEndpoint());
        replay.setEventType(original.getEventType());
        replay.setPayload(original.getPayload());
        replay.setStatus(WebhookDelivery.Status.PENDING);

        WebhookDelivery saved = deliveryRepository.save(replay);
        CompletableFuture.runAsync(() -> attemptDelivery(saved.getId()));

        return saved;
    }

    private String generateSignature(String payload, String secret) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKey = new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
            mac.init(secretKey);
            byte[] hash = mac.doFinal(payload.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate HMAC signature", e);
        }
    }

    @Scheduled(fixedRate = 60000)
    public void retryFailedDeliveries() {
        List<WebhookDelivery> pendingRetries = deliveryRepository
                .findDueForRetry(WebhookDelivery.Status.RETRYING, LocalDateTime.now());
        pendingRetries.forEach(d -> CompletableFuture.runAsync(() -> attemptDelivery(d.getId())));
    }
}
```

### Controller
```java
// WebhookController.java
@RestController
@RequestMapping("/api/webhooks")
@RequiredArgsConstructor
public class WebhookController {

    private final WebhookService webhookService;

    @PostMapping("/endpoints")
    public ResponseEntity<EndpointDTO> registerEndpoint(
            @RequestBody @Valid RegisterEndpointRequest request, Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(EndpointDTO.from(webhookService.registerEndpoint(request, getCurrentUserId(auth))));
    }

    @PostMapping("/endpoints/{id}/subscriptions")
    public ResponseEntity<SubscriptionDTO> addSubscription(
            @PathVariable Long id,
            @RequestBody @Valid AddSubscriptionRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(SubscriptionDTO.from(webhookService.addSubscription(id, request.getEventType())));
    }

    @GetMapping("/endpoints/{id}/deliveries")
    public ResponseEntity<Page<DeliveryDTO>> getDeliveries(
            @PathVariable Long id,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(webhookService.getDeliveries(id, PageRequest.of(page, size)));
    }

    @PostMapping("/deliveries/{id}/replay")
    public ResponseEntity<DeliveryDTO> replayDelivery(@PathVariable Long id) {
        return ResponseEntity.ok(DeliveryDTO.from(webhookService.replayDelivery(id)));
    }

    @GetMapping("/endpoints/{id}/deliveries/{deliveryId}/attempts")
    public ResponseEntity<List<AttemptDTO>> getAttempts(@PathVariable Long deliveryId) {
        return ResponseEntity.ok(webhookService.getDeliveryAttempts(deliveryId));
    }
}
```

---

## โปรเจค 34: OAuth2 Authorization Server

### ภาพรวม
OAuth2 Authorization Server แบบกำหนดเองโดยใช้ Spring Authorization Server รองรับการลงทะเบียน Client, PKCE, Token Introspection, Token Revocation, JWKS Endpoint และ Authorization Code Flow ครบถ้วน

### Dependencies
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-authorization-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

### Configuration
```java
// AuthorizationServerConfig.java
@Configuration
@Import(OAuth2AuthorizationServerConfiguration.class)
public class AuthorizationServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
                .oidc(Customizer.withDefaults());

        http.exceptionHandling(exceptions ->
                exceptions.defaultAuthenticationEntryPointFor(
                        new LoginUrlAuthenticationEntryPoint("/login"),
                        new MediaTypeRequestMatcher(MediaType.TEXT_HTML)));

        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth ->
                auth.requestMatchers("/api/clients/**").hasRole("ADMIN")
                    .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository(JdbcTemplate jdbcTemplate) {
        return new JdbcRegisteredClientRepository(jdbcTemplate);
    }

    @Bean
    public OAuth2AuthorizationService authorizationService(JdbcTemplate jdbcTemplate,
                                                             RegisteredClientRepository repo) {
        return new JdbcOAuth2AuthorizationService(jdbcTemplate, repo);
    }

    @Bean
    public OAuth2AuthorizationConsentService authorizationConsentService(
            JdbcTemplate jdbcTemplate, RegisteredClientRepository repo) {
        return new JdbcOAuth2AuthorizationConsentService(jdbcTemplate, repo);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() throws Exception {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
        keyPairGenerator.initialize(2048);
        KeyPair keyPair = keyPairGenerator.generateKeyPair();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();

        RSAKey rsaKey = new RSAKey.Builder(publicKey)
                .privateKey(privateKey)
                .keyID(UUID.randomUUID().toString())
                .build();

        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }

    @Bean
    public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
                .issuer("https://auth.example.com")
                .build();
    }
}
```

### Client Management Service
```java
// ClientManagementService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ClientManagementService {

    private final RegisteredClientRepository clientRepository;
    private final PasswordEncoder passwordEncoder;

    public RegisteredClient createClient(CreateClientRequest request) {
        String clientId = UUID.randomUUID().toString();
        String clientSecret = generateSecret();

        RegisteredClient.Builder builder = RegisteredClient.withId(UUID.randomUUID().toString())
                .clientId(clientId)
                .clientName(request.getName())
                .clientSecret(passwordEncoder.encode(clientSecret));

        // Set authentication methods
        request.getAuthMethods().forEach(method ->
                builder.clientAuthenticationMethod(new ClientAuthenticationMethod(method)));

        // Set authorization grant types
        request.getGrantTypes().forEach(grantType ->
                builder.authorizationGrantType(new AuthorizationGrantType(grantType)));

        // Set redirect URIs
        request.getRedirectUris().forEach(builder::redirectUri);

        // Set scopes
        request.getScopes().forEach(builder::scope);

        // Token settings
        builder.tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofHours(request.getAccessTokenTtlHours()))
                .refreshTokenTimeToLive(Duration.ofDays(request.getRefreshTokenTtlDays()))
                .reuseRefreshTokens(false)
                .build());

        // Client settings - enable PKCE if requested
        builder.clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(request.isRequireConsent())
                .requireProofKey(request.isPkceRequired())
                .build());

        RegisteredClient client = builder.build();
        clientRepository.save(client);

        log.info("Created OAuth2 client: {} ({})", request.getName(), clientId);
        return client;
    }

    private String generateSecret() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

### Controller
```java
// ClientManagementController.java
@RestController
@RequestMapping("/api/clients")
@RequiredArgsConstructor
@PreAuthorize("hasRole('ADMIN')")
public class ClientManagementController {

    private final ClientManagementService clientService;

    @PostMapping
    public ResponseEntity<ClientDTO> createClient(@RequestBody @Valid CreateClientRequest request) {
        RegisteredClient client = clientService.createClient(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(ClientDTO.from(client));
    }

    @GetMapping("/{clientId}")
    public ResponseEntity<ClientDTO> getClient(@PathVariable String clientId) {
        RegisteredClient client = clientService.findByClientId(clientId);
        return ResponseEntity.ok(ClientDTO.from(client));
    }

    @DeleteMapping("/{clientId}")
    public ResponseEntity<Void> deleteClient(@PathVariable String clientId) {
        clientService.deleteClient(clientId);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/{clientId}/rotate-secret")
    public ResponseEntity<Map<String, String>> rotateSecret(@PathVariable String clientId) {
        String newSecret = clientService.rotateSecret(clientId);
        return ResponseEntity.ok(Map.of("clientSecret", newSecret));
    }
}
```

---

## โปรเจค 35: API Rate Limiter Service

### ภาพรวม
บริการจำกัดอัตราการเรียก API (Rate Limiting) ที่รองรับ Policy แบบต่างๆ (ต่อ User/IP/API Key), อัลกอริทึม Token Bucket และ Sliding Window, HTTP Headers (X-RateLimit-*), การจัดเก็บด้วย Redis และ Bypass Rules สำหรับกรณีพิเศษ

### Flyway Migration
```sql
-- V1__create_rate_limiter_tables.sql
CREATE TABLE rate_limit_policies (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    key_type VARCHAR(20) NOT NULL,
    algorithm VARCHAR(20) DEFAULT 'TOKEN_BUCKET',
    limit_requests INTEGER NOT NULL,
    window_seconds INTEGER NOT NULL,
    burst_size INTEGER,
    enabled BOOLEAN DEFAULT TRUE,
    priority INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE bypass_rules (
    id BIGSERIAL PRIMARY KEY,
    policy_id BIGINT REFERENCES rate_limit_policies(id) ON DELETE CASCADE,
    rule_type VARCHAR(20) NOT NULL,
    value VARCHAR(200) NOT NULL,
    reason VARCHAR(500),
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE rate_limit_violations (
    id BIGSERIAL PRIMARY KEY,
    policy_id BIGINT REFERENCES rate_limit_policies(id),
    key_value VARCHAR(200) NOT NULL,
    endpoint VARCHAR(500),
    ip_address VARCHAR(45),
    violated_at TIMESTAMP DEFAULT NOW()
);
```

### Service
```java
// RateLimiterService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class RateLimiterService {

    private final RateLimitPolicyRepository policyRepository;
    private final BypassRuleRepository bypassRepository;
    private final RedisTemplate<String, String> redisTemplate;

    private static final String RATE_LIMIT_PREFIX = "rl:";

    public RateLimitResult checkRateLimit(RateLimitRequest request) {
        RateLimitPolicy policy = policyRepository.findApplicablePolicy(
                request.getKeyType(), request.getEndpoint())
                .orElse(null);

        if (policy == null || !policy.getEnabled()) {
            return RateLimitResult.allowed();
        }

        // Check bypass rules
        if (isBypassed(request, policy)) {
            return RateLimitResult.allowed();
        }

        String key = buildKey(policy, request);

        return switch (policy.getAlgorithm()) {
            case "TOKEN_BUCKET" -> checkTokenBucket(key, policy);
            case "SLIDING_WINDOW" -> checkSlidingWindow(key, policy);
            case "FIXED_WINDOW" -> checkFixedWindow(key, policy);
            default -> checkSlidingWindow(key, policy);
        };
    }

    private RateLimitResult checkTokenBucket(String key, RateLimitPolicy policy) {
        String bucketKey = RATE_LIMIT_PREFIX + "tb:" + key;
        int burstSize = policy.getBurstSize() != null ? policy.getBurstSize() : policy.getLimitRequests();

        List<Object> results = redisTemplate.execute(new SessionCallback<List<Object>>() {
            @Override
            public List<Object> execute(RedisOperations operations) throws DataAccessException {
                operations.multi();
                operations.opsForValue().increment(bucketKey + ":tokens", -1);
                operations.expire(bucketKey + ":tokens", Duration.ofSeconds(policy.getWindowSeconds()));
                return operations.exec();
            }
        });

        long currentTokens = Long.parseLong(redisTemplate.opsForValue()
                .getAndExpire(bucketKey + ":tokens", Duration.ofSeconds(policy.getWindowSeconds())) != null
                ? redisTemplate.opsForValue().get(bucketKey + ":tokens") : "0");

        if (currentTokens < 0) {
            // Refill logic
            redisTemplate.opsForValue().set(bucketKey + ":tokens",
                    String.valueOf(burstSize - 1),
                    Duration.ofSeconds(policy.getWindowSeconds()));
        }

        long remaining = Math.max(0, policy.getLimitRequests() - Math.abs(currentTokens));
        long resetAt = System.currentTimeMillis() / 1000 + policy.getWindowSeconds();

        boolean allowed = currentTokens >= 0;
        return new RateLimitResult(allowed, policy.getLimitRequests(), remaining, resetAt,
                policy.getWindowSeconds());
    }

    private RateLimitResult checkSlidingWindow(String key, RateLimitPolicy policy) {
        String windowKey = RATE_LIMIT_PREFIX + "sw:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - (policy.getWindowSeconds() * 1000L);

        // Remove old entries
        redisTemplate.opsForZSet().removeRangeByScore(windowKey, 0, windowStart);

        // Count current requests
        Long count = redisTemplate.opsForZSet().size(windowKey);
        long currentCount = count != null ? count : 0;

        if (currentCount >= policy.getLimitRequests()) {
            long resetAt = System.currentTimeMillis() / 1000 + policy.getWindowSeconds();
            return RateLimitResult.denied(policy.getLimitRequests(), 0, resetAt, policy.getWindowSeconds());
        }

        // Add current request
        redisTemplate.opsForZSet().add(windowKey, String.valueOf(now), now);
        redisTemplate.expire(windowKey, Duration.ofSeconds(policy.getWindowSeconds() + 1));

        long remaining = policy.getLimitRequests() - currentCount - 1;
        long resetAt = System.currentTimeMillis() / 1000 + policy.getWindowSeconds();

        return RateLimitResult.allowed(policy.getLimitRequests(), remaining, resetAt, policy.getWindowSeconds());
    }

    private boolean isBypassed(RateLimitRequest request, RateLimitPolicy policy) {
        return bypassRepository.existsByPolicyAndValue(
                policy.getId(), request.getKeyValue(), LocalDateTime.now());
    }

    private String buildKey(RateLimitPolicy policy, RateLimitRequest request) {
        return policy.getId() + ":" + request.getKeyType() + ":" + request.getKeyValue();
    }
}

// RateLimitFilter.java
@Component
@RequiredArgsConstructor
@Order(1)
public class RateLimitFilter extends OncePerRequestFilter {

    private final RateLimiterService rateLimiterService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        String ipAddress = extractIpAddress(request);
        String apiKey = request.getHeader("X-API-Key");
        String userId = extractUserId(request);

        // Determine key type and value
        String keyType = "IP";
        String keyValue = ipAddress;

        if (apiKey != null) {
            keyType = "API_KEY";
            keyValue = apiKey;
        } else if (userId != null) {
            keyType = "USER";
            keyValue = userId;
        }

        RateLimitRequest rateLimitRequest = new RateLimitRequest(
                keyType, keyValue, request.getRequestURI(), ipAddress);

        RateLimitResult result = rateLimiterService.checkRateLimit(rateLimitRequest);

        // Add rate limit headers
        response.setHeader("X-RateLimit-Limit", String.valueOf(result.getLimit()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(result.getRemaining()));
        response.setHeader("X-RateLimit-Reset", String.valueOf(result.getResetAt()));
        response.setHeader("X-RateLimit-Window", String.valueOf(result.getWindowSeconds()));

        if (!result.isAllowed()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setHeader("Retry-After", String.valueOf(result.getResetAt() - System.currentTimeMillis() / 1000));

            ObjectMapper mapper = new ObjectMapper();
            response.getWriter().write(mapper.writeValueAsString(Map.of(
                    "error", "Rate limit exceeded",
                    "message", "Too many requests. Please retry after " + result.getResetAt(),
                    "retryAfter", result.getResetAt()
            )));
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

### Controller
```java
// RateLimitController.java
@RestController
@RequestMapping("/api/rate-limits")
@RequiredArgsConstructor
public class RateLimitController {

    private final RateLimiterService rateLimiterService;

    @PostMapping("/policies")
    public ResponseEntity<PolicyDTO> createPolicy(@RequestBody @Valid CreatePolicyRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(PolicyDTO.from(rateLimiterService.createPolicy(request)));
    }

    @GetMapping("/policies")
    public ResponseEntity<List<PolicyDTO>> listPolicies() {
        return ResponseEntity.ok(rateLimiterService.listPolicies());
    }

    @PostMapping("/policies/{id}/bypass")
    public ResponseEntity<BypassRuleDTO> addBypassRule(
            @PathVariable Long id,
            @RequestBody @Valid AddBypassRuleRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(BypassRuleDTO.from(rateLimiterService.addBypassRule(id, request)));
    }

    @GetMapping("/status/{keyType}/{keyValue}")
    public ResponseEntity<RateLimitStatusDTO> getRateLimitStatus(
            @PathVariable String keyType,
            @PathVariable String keyValue) {
        return ResponseEntity.ok(rateLimiterService.getRateLimitStatus(keyType, keyValue));
    }

    @DeleteMapping("/reset/{keyType}/{keyValue}")
    public ResponseEntity<Void> resetRateLimit(
            @PathVariable String keyType,
            @PathVariable String keyValue) {
        rateLimiterService.resetRateLimit(keyType, keyValue);
        return ResponseEntity.noContent().build();
    }
}
```

### Docker Compose
```yaml
# docker-compose.yml (Part 107)
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/devtoolsdb
      SPRING_REDIS_HOST: redis
      GEOIP_DB_PATH: /data/GeoLite2-City.mmdb
    volumes:
      - ./geoip:/data
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: devtoolsdb
      POSTGRES_USER: devtools
      POSTGRES_PASSWORD: devpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## สรุป Part 107

| โปรเจค | เทคโนโลยีหลัก | ความซับซ้อน |
|--------|--------------|------------|
| 31. URL Shortener | Redis Cache, GeoIP, ZXing | กลาง |
| 32. Feature Flags | Redis, Targeting Rules, SDK | สูง |
| 33. Webhook Platform | Retry Logic, HMAC, Async | สูง |
| 34. OAuth2 Server | Spring Authorization Server, PKCE | สูงมาก |
| 35. Rate Limiter | Redis, Token Bucket, Sliding Window | สูง |

---

## Navigation

- [← Part 106: Real-time Systems](part-106-realtime-systems.md)
- [Part 108: Content & Document →](part-108-content-document.md)
- [กลับหน้าหลัก](README.md)

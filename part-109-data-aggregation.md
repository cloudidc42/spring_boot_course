# Part 109: โปรเจค 41-45 — Data & Aggregation Services

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 41-45

---

## โปรเจค 41: Social Media API

### ภาพรวม
API สำหรับโซเชียลมีเดียที่รองรับผู้ใช้ (Users), โพสต์ (Posts), การกดถูกใจ (Likes), ความคิดเห็น (Comments), การติดตาม (Follows), อัลกอริทึม Feed (Chronological + Ranked), Stories, Hashtags และการแจ้งเตือน

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
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

### Flyway Migration
```sql
-- V1__create_social_media_tables.sql
CREATE TABLE social_users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    website VARCHAR(200),
    follower_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    post_count INTEGER DEFAULT 0,
    is_verified BOOLEAN DEFAULT FALSE,
    is_private BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE follows (
    id BIGSERIAL PRIMARY KEY,
    follower_id BIGINT REFERENCES social_users(id) ON DELETE CASCADE,
    following_id BIGINT REFERENCES social_users(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(follower_id, following_id)
);

CREATE TABLE hashtags (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    post_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    author_id BIGINT REFERENCES social_users(id) ON DELETE CASCADE,
    content TEXT,
    media_urls JSONB,
    post_type VARCHAR(20) DEFAULT 'POST',
    visibility VARCHAR(20) DEFAULT 'PUBLIC',
    like_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    share_count INTEGER DEFAULT 0,
    view_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE post_hashtags (
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    hashtag_id BIGINT REFERENCES hashtags(id) ON DELETE CASCADE,
    PRIMARY KEY(post_id, hashtag_id)
);

CREATE TABLE post_likes (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES social_users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(post_id, user_id)
);

CREATE TABLE post_comments (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT REFERENCES posts(id) ON DELETE CASCADE,
    author_id BIGINT REFERENCES social_users(id),
    parent_id BIGINT REFERENCES post_comments(id),
    content TEXT NOT NULL,
    like_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stories (
    id BIGSERIAL PRIMARY KEY,
    author_id BIGINT REFERENCES social_users(id) ON DELETE CASCADE,
    media_url VARCHAR(500) NOT NULL,
    media_type VARCHAR(20) DEFAULT 'IMAGE',
    caption TEXT,
    view_count INTEGER DEFAULT 0,
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '24 hours',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_posts_author_created ON posts(author_id, created_at DESC);
CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);
```

### Entity Classes
```java
// Post.java
@Entity
@Table(name = "posts")
@Data
@NoArgsConstructor
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private SocialUser author;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Type(JsonType.class)
    @Column(name = "media_urls", columnDefinition = "jsonb")
    private List<String> mediaUrls;

    @Enumerated(EnumType.STRING)
    @Column(name = "post_type")
    private PostType postType = PostType.POST;

    @Enumerated(EnumType.STRING)
    private Visibility visibility = Visibility.PUBLIC;

    @Column(name = "like_count")
    private Integer likeCount = 0;

    @Column(name = "comment_count")
    private Integer commentCount = 0;

    @Column(name = "share_count")
    private Integer shareCount = 0;

    @Column(name = "view_count")
    private Integer viewCount = 0;

    @ManyToMany
    @JoinTable(name = "post_hashtags",
            joinColumns = @JoinColumn(name = "post_id"),
            inverseJoinColumns = @JoinColumn(name = "hashtag_id"))
    private Set<Hashtag> hashtags = new HashSet<>();

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    public enum PostType { POST, REEL, QUOTE }
    public enum Visibility { PUBLIC, FOLLOWERS_ONLY, PRIVATE }
}
```

### Service
```java
// FeedService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class FeedService {

    private final PostRepository postRepository;
    private final FollowRepository followRepository;
    private final PostLikeRepository likeRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final HashtagRepository hashtagRepository;

    private static final String FEED_CACHE_KEY = "feed:user:";
    private static final int FEED_SIZE = 50;

    public Page<Post> getChronologicalFeed(Long userId, Pageable pageable) {
        // Get list of users the current user follows
        List<Long> followingIds = followRepository.findFollowingIds(userId);
        followingIds.add(userId); // Include own posts

        return postRepository.findByAuthorIdInAndVisibilityOrderByCreatedAtDesc(
                followingIds, Post.Visibility.PUBLIC, pageable);
    }

    public List<PostDTO> getRankedFeed(Long userId, int page) {
        // Get following list
        List<Long> followingIds = followRepository.findFollowingIds(userId);

        // Fetch recent posts from following
        List<Post> candidatePosts = postRepository.findRecentPostsByAuthors(
                followingIds, LocalDateTime.now().minusDays(7),
                PageRequest.of(0, FEED_SIZE * 3));

        // Score each post
        List<ScoredPost> scoredPosts = candidatePosts.stream()
                .map(post -> new ScoredPost(post, calculateEngagementScore(post, userId)))
                .sorted(Comparator.comparingDouble(ScoredPost::getScore).reversed())
                .collect(Collectors.toList());

        // Paginate
        int start = page * FEED_SIZE;
        int end = Math.min(start + FEED_SIZE, scoredPosts.size());

        return scoredPosts.subList(start, end).stream()
                .map(sp -> PostDTO.from(sp.getPost()))
                .collect(Collectors.toList());
    }

    private double calculateEngagementScore(Post post, Long userId) {
        double score = 0;

        // Time decay (newer posts score higher)
        long minutesAgo = ChronoUnit.MINUTES.between(post.getCreatedAt(), LocalDateTime.now());
        double timeDecay = 1.0 / (1 + Math.log1p(minutesAgo / 60.0));

        // Engagement metrics
        double engagementScore = (post.getLikeCount() * 1.0)
                + (post.getCommentCount() * 3.0)
                + (post.getShareCount() * 5.0);

        // Relationship strength (closer follows rank higher)
        double relationshipScore = followRepository.isCloseFriend(userId, post.getAuthor().getId()) ? 2.0 : 1.0;

        score = (engagementScore + 1) * timeDecay * relationshipScore;
        return score;
    }

    public Post createPost(Long authorId, CreatePostRequest request) {
        SocialUser author = userRepository.findById(authorId)
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        Post post = new Post();
        post.setAuthor(author);
        post.setContent(request.getContent());
        post.setMediaUrls(request.getMediaUrls());
        post.setPostType(request.getPostType());
        post.setVisibility(request.getVisibility());

        // Extract and process hashtags
        Set<String> hashtagNames = extractHashtags(request.getContent());
        Set<Hashtag> hashtags = hashtagNames.stream()
                .map(name -> hashtagRepository.findByName(name.toLowerCase())
                        .orElseGet(() -> {
                            Hashtag ht = new Hashtag();
                            ht.setName(name.toLowerCase());
                            return hashtagRepository.save(ht);
                        }))
                .collect(Collectors.toSet());
        post.setHashtags(hashtags);

        Post saved = postRepository.save(post);

        // Increment post count for user
        userRepository.incrementPostCount(authorId);

        // Increment hashtag post counts
        hashtagNames.forEach(name -> hashtagRepository.incrementPostCount(name.toLowerCase()));

        // Invalidate follower feeds
        invalidateFollowerFeeds(authorId);

        return saved;
    }

    public void likePost(Long postId, Long userId) {
        if (likeRepository.existsByPostIdAndUserId(postId, userId)) {
            // Unlike
            likeRepository.deleteByPostIdAndUserId(postId, userId);
            postRepository.decrementLikeCount(postId);
        } else {
            // Like
            PostLike like = new PostLike();
            like.setPost(new Post(postId));
            like.setUser(new SocialUser(userId));
            likeRepository.save(like);
            postRepository.incrementLikeCount(postId);
        }
    }

    public List<PostDTO> getTrendingHashtagPosts(String hashtag, Pageable pageable) {
        return postRepository.findByHashtagName(hashtag.toLowerCase(), pageable)
                .stream().map(PostDTO::from).collect(Collectors.toList());
    }

    private Set<String> extractHashtags(String content) {
        if (content == null) return Collections.emptySet();
        Set<String> hashtags = new HashSet<>();
        Pattern pattern = Pattern.compile("#(\\w+)");
        Matcher matcher = pattern.matcher(content);
        while (matcher.find()) {
            hashtags.add(matcher.group(1));
        }
        return hashtags;
    }
}
```

### Controller
```java
// SocialController.java
@RestController
@RequestMapping("/api/social")
@RequiredArgsConstructor
public class SocialController {

    private final FeedService feedService;
    private final PostService postService;

    @PostMapping("/posts")
    public ResponseEntity<PostDTO> createPost(@RequestBody @Valid CreatePostRequest request,
                                               Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(PostDTO.from(feedService.createPost(getCurrentUserId(auth), request)));
    }

    @GetMapping("/feed")
    public ResponseEntity<Page<PostDTO>> getFeed(
            @RequestParam(defaultValue = "CHRONOLOGICAL") String type,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            Authentication auth) {
        Long userId = getCurrentUserId(auth);
        if ("RANKED".equals(type)) {
            return ResponseEntity.ok(new PageImpl<>(feedService.getRankedFeed(userId, page)));
        }
        return ResponseEntity.ok(feedService.getChronologicalFeed(userId, PageRequest.of(page, size))
                .map(PostDTO::from));
    }

    @PostMapping("/posts/{id}/like")
    public ResponseEntity<Void> likePost(@PathVariable Long id, Authentication auth) {
        feedService.likePost(id, getCurrentUserId(auth));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/users/{id}/follow")
    public ResponseEntity<Void> followUser(@PathVariable Long id, Authentication auth) {
        postService.followUser(getCurrentUserId(auth), id);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/hashtags/{name}/posts")
    public ResponseEntity<Page<PostDTO>> getHashtagPosts(
            @PathVariable String name,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(feedService.getTrendingHashtagPosts(name,
                PageRequest.of(page, size)).stream()
                .collect(Collectors.collectingAndThen(
                        Collectors.toList(),
                        list -> new PageImpl<>(list))));
    }
}
```

---

## โปรเจค 42: News Aggregator

### ภาพรวม
บริการรวบรวมข่าว (News Aggregator) จากหลายแหล่ง RSS/API, จัดการบทความ, หมวดหมู่, บันทึกสำหรับอ่านภายหลัง (Read Later), Feed ที่ปรับแต่งตามประวัติการอ่าน (Personalized Feed) และหัวข้อที่กำลังมาแรง (Trending Topics)

### Flyway Migration
```sql
-- V1__create_news_tables.sql
CREATE TABLE news_sources (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    url VARCHAR(500) NOT NULL,
    feed_url VARCHAR(500),
    source_type VARCHAR(20) DEFAULT 'RSS',
    categories JSONB,
    language VARCHAR(10) DEFAULT 'th',
    is_active BOOLEAN DEFAULT TRUE,
    last_fetched_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE news_categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT
);

CREATE TABLE articles (
    id BIGSERIAL PRIMARY KEY,
    source_id BIGINT REFERENCES news_sources(id),
    category_id BIGINT REFERENCES news_categories(id),
    title VARCHAR(500) NOT NULL,
    summary TEXT,
    content TEXT,
    url VARCHAR(1000) UNIQUE NOT NULL,
    image_url VARCHAR(500),
    author VARCHAR(200),
    published_at TIMESTAMP,
    fetched_at TIMESTAMP DEFAULT NOW(),
    view_count INTEGER DEFAULT 0,
    share_count INTEGER DEFAULT 0,
    tags JSONB
);

CREATE TABLE user_reading_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    article_id BIGINT REFERENCES articles(id) ON DELETE CASCADE,
    read_at TIMESTAMP DEFAULT NOW(),
    read_duration_seconds INTEGER,
    UNIQUE(user_id, article_id)
);

CREATE TABLE user_read_later (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    article_id BIGINT REFERENCES articles(id) ON DELETE CASCADE,
    saved_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, article_id)
);

CREATE TABLE trending_topics (
    id BIGSERIAL PRIMARY KEY,
    topic VARCHAR(200) NOT NULL,
    article_count INTEGER DEFAULT 0,
    score DOUBLE PRECISION DEFAULT 0,
    period VARCHAR(20) DEFAULT 'HOUR',
    calculated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_articles_category ON articles(category_id, published_at DESC);
CREATE INDEX idx_articles_published ON articles(published_at DESC);
CREATE INDEX idx_reading_history_user ON user_reading_history(user_id, read_at DESC);
```

### Service
```java
// NewsAggregatorService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class NewsAggregatorService {

    private final NewsSourceRepository sourceRepository;
    private final ArticleRepository articleRepository;
    private final ReadingHistoryRepository historyRepository;
    private final ReadLaterRepository readLaterRepository;
    private final RssService rssService;
    private final NewsApiService newsApiService;

    @Scheduled(fixedRate = 1800000) // Every 30 minutes
    public void fetchArticles() {
        List<NewsSource> sources = sourceRepository.findByIsActive(true);
        sources.parallelStream().forEach(this::fetchFromSource);
    }

    private void fetchFromSource(NewsSource source) {
        try {
            List<Article> articles = switch (source.getSourceType()) {
                case "RSS" -> rssService.fetchArticles(source.getFeedUrl());
                case "API" -> newsApiService.fetchArticles(source);
                default -> Collections.emptyList();
            };

            articles.forEach(article -> {
                article.setSource(source);
                if (!articleRepository.existsByUrl(article.getUrl())) {
                    articleRepository.save(article);
                }
            });

            source.setLastFetchedAt(LocalDateTime.now());
            sourceRepository.save(source);
            log.info("Fetched {} articles from {}", articles.size(), source.getName());

        } catch (Exception e) {
            log.error("Failed to fetch from source {}: {}", source.getName(), e.getMessage());
        }
    }

    public Page<Article> getPersonalizedFeed(Long userId, Pageable pageable) {
        // Get user's reading history to determine interests
        List<Long> recentlyReadCategoryIds = historyRepository
                .findTopCategoriesByUserId(userId, 10);

        if (recentlyReadCategoryIds.isEmpty()) {
            // New user: return latest articles
            return articleRepository.findAllOrderByPublishedAtDesc(pageable);
        }

        // Get articles from preferred categories they haven't read
        List<Long> readArticleIds = historyRepository.findReadArticleIds(userId);

        return articleRepository.findPersonalizedFeed(
                recentlyReadCategoryIds, readArticleIds, pageable);
    }

    public void recordReading(Long userId, Long articleId, Integer durationSeconds) {
        ReadingHistory history = readLaterRepository.findByUserIdAndArticleId(userId, articleId)
                .map(rl -> {
                    ReadingHistory h = new ReadingHistory();
                    h.setUserId(userId);
                    h.setArticleId(articleId);
                    return h;
                })
                .orElseGet(() -> {
                    ReadingHistory h = new ReadingHistory();
                    h.setUserId(userId);
                    h.setArticleId(articleId);
                    return h;
                });

        history.setReadDurationSeconds(durationSeconds);

        try {
            historyRepository.save(history);
        } catch (DataIntegrityViolationException e) {
            // Already recorded, update duration
            historyRepository.updateDuration(userId, articleId, durationSeconds);
        }

        articleRepository.incrementViewCount(articleId);
    }

    public List<TrendingTopic> getTrendingTopics(String period) {
        return trendingTopicRepository.findByPeriodOrderByScoreDesc(period,
                PageRequest.of(0, 10)).getContent();
    }

    @Scheduled(fixedRate = 3600000)
    public void calculateTrendingTopics() {
        LocalDateTime from = LocalDateTime.now().minusHours(1);
        List<Object[]> tagCounts = articleRepository.countTagOccurrencesSince(from);

        trendingTopicRepository.deleteByPeriod("HOUR");
        tagCounts.forEach(row -> {
            TrendingTopic topic = new TrendingTopic();
            topic.setTopic((String) row[0]);
            topic.setArticleCount(((Number) row[1]).intValue());
            topic.setScore(((Number) row[1]).doubleValue());
            topic.setPeriod("HOUR");
            trendingTopicRepository.save(topic);
        });
    }
}

// RssService.java
@Service
@RequiredArgsConstructor
public class RssService {

    private final RestTemplate restTemplate;

    public List<Article> fetchArticles(String feedUrl) throws Exception {
        String xmlContent = restTemplate.getForObject(feedUrl, String.class);
        if (xmlContent == null) return Collections.emptyList();

        DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
        DocumentBuilder db = dbf.newDocumentBuilder();
        org.w3c.dom.Document doc = db.parse(new InputSource(new StringReader(xmlContent)));

        NodeList items = doc.getElementsByTagName("item");
        List<Article> articles = new ArrayList<>();

        for (int i = 0; i < items.getLength(); i++) {
            org.w3c.dom.Element item = (org.w3c.dom.Element) items.item(i);
            Article article = new Article();
            article.setTitle(getTextContent(item, "title"));
            article.setSummary(getTextContent(item, "description"));
            article.setUrl(getTextContent(item, "link"));
            article.setAuthor(getTextContent(item, "author"));

            String pubDateStr = getTextContent(item, "pubDate");
            if (pubDateStr != null) {
                article.setPublishedAt(parseRssDate(pubDateStr));
            }

            // Try to get image from media:content or enclosure
            article.setImageUrl(extractImageUrl(item));
            articles.add(article);
        }

        return articles;
    }
}
```

### Controller
```java
// NewsController.java
@RestController
@RequestMapping("/api/news")
@RequiredArgsConstructor
public class NewsController {

    private final NewsAggregatorService newsService;

    @GetMapping("/feed")
    public ResponseEntity<Page<ArticleDTO>> getFeed(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            Authentication auth) {
        Long userId = auth != null ? getCurrentUserId(auth) : null;
        Page<Article> feed = userId != null
                ? newsService.getPersonalizedFeed(userId, PageRequest.of(page, size))
                : newsService.getLatestArticles(PageRequest.of(page, size));
        return ResponseEntity.ok(feed.map(ArticleDTO::from));
    }

    @GetMapping("/categories/{slug}")
    public ResponseEntity<Page<ArticleDTO>> getByCategory(
            @PathVariable String slug,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(newsService.getByCategory(slug, PageRequest.of(page, size))
                .map(ArticleDTO::from));
    }

    @PostMapping("/articles/{id}/read")
    public ResponseEntity<Void> markAsRead(@PathVariable Long id,
                                            @RequestBody ReadingProgressRequest request,
                                            Authentication auth) {
        newsService.recordReading(getCurrentUserId(auth), id, request.getDurationSeconds());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/articles/{id}/read-later")
    public ResponseEntity<Void> saveForLater(@PathVariable Long id, Authentication auth) {
        newsService.saveForLater(getCurrentUserId(auth), id);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/trending")
    public ResponseEntity<List<TrendingTopicDTO>> getTrending(
            @RequestParam(defaultValue = "HOUR") String period) {
        return ResponseEntity.ok(newsService.getTrendingTopics(period).stream()
                .map(TrendingTopicDTO::from).collect(Collectors.toList()));
    }
}
```

---

## โปรเจค 43: Weather API Aggregator

### ภาพรวม
บริการรวมข้อมูลสภาพอากาศจากหลายผู้ให้บริการ (Multiple Weather Providers), การ Cache, Query ตามตำแหน่ง (Location-based), พยากรณ์อากาศ (Forecasts), การแจ้งเตือนสภาพอากาศ (Weather Alerts) และข้อมูลย้อนหลัง (Historical Data)

### Flyway Migration
```sql
-- V1__create_weather_tables.sql
CREATE TABLE weather_providers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    api_key_encrypted VARCHAR(500),
    base_url VARCHAR(200),
    priority INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE locations (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    country_code VARCHAR(3),
    state VARCHAR(100),
    city VARCHAR(100),
    lat DECIMAL(10,8) NOT NULL,
    lng DECIMAL(11,8) NOT NULL,
    timezone VARCHAR(50),
    UNIQUE(lat, lng)
);

CREATE TABLE weather_cache (
    id BIGSERIAL PRIMARY KEY,
    location_id BIGINT REFERENCES locations(id),
    provider_id BIGINT REFERENCES weather_providers(id),
    data_type VARCHAR(20) NOT NULL,
    data JSONB NOT NULL,
    cached_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    UNIQUE(location_id, provider_id, data_type)
);

CREATE TABLE weather_history (
    id BIGSERIAL PRIMARY KEY,
    location_id BIGINT REFERENCES locations(id),
    temperature_min DECIMAL(5,2),
    temperature_max DECIMAL(5,2),
    temperature_avg DECIMAL(5,2),
    humidity INTEGER,
    precipitation DECIMAL(6,2),
    wind_speed DECIMAL(5,2),
    condition VARCHAR(50),
    recorded_date DATE,
    UNIQUE(location_id, recorded_date)
);

CREATE TABLE weather_alerts (
    id BIGSERIAL PRIMARY KEY,
    location_id BIGINT REFERENCES locations(id),
    alert_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20),
    headline VARCHAR(500),
    description TEXT,
    starts_at TIMESTAMP,
    ends_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Service
```java
// WeatherAggregatorService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class WeatherAggregatorService {

    private final WeatherProviderRepository providerRepository;
    private final WeatherCacheRepository cacheRepository;
    private final LocationRepository locationRepository;
    private final OpenWeatherService openWeatherService;
    private final WeatherApiService weatherApiService;

    private static final Duration CURRENT_WEATHER_TTL = Duration.ofMinutes(15);
    private static final Duration FORECAST_TTL = Duration.ofHours(1);

    public CurrentWeather getCurrentWeather(double lat, double lng) {
        Location location = findOrCreateLocation(lat, lng);

        // Check cache
        Optional<WeatherCache> cached = cacheRepository.findValidCache(
                location.getId(), "CURRENT", LocalDateTime.now());

        if (cached.isPresent()) {
            return deserialize(cached.get().getData(), CurrentWeather.class);
        }

        // Fetch from provider with fallback
        CurrentWeather weather = fetchCurrentWithFallback(location);

        // Cache result
        cacheWeather(location, "CURRENT", weather, CURRENT_WEATHER_TTL);

        return weather;
    }

    public WeatherForecast getForecast(double lat, double lng, int days) {
        Location location = findOrCreateLocation(lat, lng);

        Optional<WeatherCache> cached = cacheRepository.findValidCache(
                location.getId(), "FORECAST_" + days, LocalDateTime.now());

        if (cached.isPresent()) {
            return deserialize(cached.get().getData(), WeatherForecast.class);
        }

        WeatherForecast forecast = fetchForecastWithFallback(location, days);
        cacheWeather(location, "FORECAST_" + days, forecast, FORECAST_TTL);

        return forecast;
    }

    private CurrentWeather fetchCurrentWithFallback(Location location) {
        List<WeatherProvider> providers = providerRepository.findByIsActiveOrderByPriority(true);

        for (WeatherProvider provider : providers) {
            try {
                return switch (provider.getName()) {
                    case "openweathermap" -> openWeatherService.getCurrentWeather(
                            location.getLat().doubleValue(),
                            location.getLng().doubleValue());
                    case "weatherapi" -> weatherApiService.getCurrentWeather(
                            location.getLat().doubleValue(),
                            location.getLng().doubleValue());
                    default -> throw new UnsupportedOperationException("Unknown provider: " + provider.getName());
                };
            } catch (Exception e) {
                log.warn("Provider {} failed: {}, trying next...", provider.getName(), e.getMessage());
            }
        }

        throw new ServiceUnavailableException("All weather providers are unavailable");
    }

    public List<WeatherAlert> getAlerts(double lat, double lng) {
        Location location = findOrCreateLocation(lat, lng);
        return alertRepository.findActiveAlertsForLocation(
                location.getId(), LocalDateTime.now());
    }

    public List<WeatherHistoryRecord> getHistoricalData(double lat, double lng,
                                                          LocalDate from, LocalDate to) {
        Location location = findOrCreateLocation(lat, lng);
        return weatherHistoryRepository.findByLocationAndDateRange(location.getId(), from, to);
    }

    private Location findOrCreateLocation(double lat, double lng) {
        BigDecimal bdLat = BigDecimal.valueOf(lat).setScale(4, RoundingMode.HALF_UP);
        BigDecimal bdLng = BigDecimal.valueOf(lng).setScale(4, RoundingMode.HALF_UP);

        return locationRepository.findByLatAndLng(bdLat, bdLng)
                .orElseGet(() -> {
                    Location loc = new Location();
                    loc.setLat(bdLat);
                    loc.setLng(bdLng);
                    // Reverse geocode to get name
                    String name = reverseGeocode(lat, lng);
                    loc.setName(name);
                    return locationRepository.save(loc);
                });
    }
}
```

### Controller
```java
// WeatherController.java
@RestController
@RequestMapping("/api/weather")
@RequiredArgsConstructor
public class WeatherController {

    private final WeatherAggregatorService weatherService;

    @GetMapping("/current")
    public ResponseEntity<CurrentWeatherDTO> getCurrentWeather(
            @RequestParam double lat,
            @RequestParam double lng) {
        return ResponseEntity.ok(CurrentWeatherDTO.from(weatherService.getCurrentWeather(lat, lng)));
    }

    @GetMapping("/forecast")
    public ResponseEntity<WeatherForecastDTO> getForecast(
            @RequestParam double lat,
            @RequestParam double lng,
            @RequestParam(defaultValue = "7") int days) {
        return ResponseEntity.ok(WeatherForecastDTO.from(weatherService.getForecast(lat, lng, days)));
    }

    @GetMapping("/alerts")
    public ResponseEntity<List<WeatherAlertDTO>> getAlerts(
            @RequestParam double lat,
            @RequestParam double lng) {
        return ResponseEntity.ok(weatherService.getAlerts(lat, lng).stream()
                .map(WeatherAlertDTO::from).collect(Collectors.toList()));
    }

    @GetMapping("/history")
    public ResponseEntity<List<WeatherHistoryDTO>> getHistory(
            @RequestParam double lat,
            @RequestParam double lng,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        return ResponseEntity.ok(weatherService.getHistoricalData(lat, lng, from, to).stream()
                .map(WeatherHistoryDTO::from).collect(Collectors.toList()));
    }

    @GetMapping("/location/search")
    public ResponseEntity<List<LocationDTO>> searchLocations(@RequestParam String q) {
        return ResponseEntity.ok(weatherService.searchLocations(q));
    }
}
```

---

## โปรเจค 44: Stock Market Tracker

### ภาพรวม
ระบบติดตามตลาดหุ้นที่รองรับการจัดการ Portfolio, รายการ Watchlist, การแจ้งเตือนราคา (Price Alerts), กราฟราคาย้อนหลัง (Historical Charts), การคำนวณกำไร/ขาดทุน (P&L), ประวัติธุรกรรม (Transaction History) และการติดตามเงินปันผล (Dividend Tracking)

### Flyway Migration
```sql
-- V1__create_stock_tracker_tables.sql
CREATE TABLE stocks (
    id BIGSERIAL PRIMARY KEY,
    symbol VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    exchange VARCHAR(20),
    sector VARCHAR(100),
    industry VARCHAR(100),
    currency VARCHAR(5) DEFAULT 'USD',
    last_price DECIMAL(15,4),
    change_percent DECIMAL(8,4),
    volume BIGINT,
    market_cap BIGINT,
    last_updated TIMESTAMP
);

CREATE TABLE portfolios (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    currency VARCHAR(5) DEFAULT 'USD',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE portfolio_positions (
    id BIGSERIAL PRIMARY KEY,
    portfolio_id BIGINT REFERENCES portfolios(id) ON DELETE CASCADE,
    stock_id BIGINT REFERENCES stocks(id),
    total_shares DECIMAL(15,6) DEFAULT 0,
    average_cost DECIMAL(15,4) DEFAULT 0,
    realized_pnl DECIMAL(15,2) DEFAULT 0,
    UNIQUE(portfolio_id, stock_id)
);

CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    portfolio_id BIGINT REFERENCES portfolios(id),
    stock_id BIGINT REFERENCES stocks(id),
    transaction_type VARCHAR(10) NOT NULL,
    shares DECIMAL(15,6) NOT NULL,
    price DECIMAL(15,4) NOT NULL,
    fees DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(15,2) NOT NULL,
    notes TEXT,
    transaction_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE watchlists (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE watchlist_items (
    id BIGSERIAL PRIMARY KEY,
    watchlist_id BIGINT REFERENCES watchlists(id) ON DELETE CASCADE,
    stock_id BIGINT REFERENCES stocks(id),
    alert_price_above DECIMAL(15,4),
    alert_price_below DECIMAL(15,4),
    notes TEXT,
    UNIQUE(watchlist_id, stock_id)
);

CREATE TABLE price_alerts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    stock_id BIGINT REFERENCES stocks(id),
    condition VARCHAR(20) NOT NULL,
    target_price DECIMAL(15,4) NOT NULL,
    is_triggered BOOLEAN DEFAULT FALSE,
    triggered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE dividends (
    id BIGSERIAL PRIMARY KEY,
    stock_id BIGINT REFERENCES stocks(id),
    amount DECIMAL(10,4) NOT NULL,
    currency VARCHAR(5) DEFAULT 'USD',
    ex_date DATE NOT NULL,
    payment_date DATE,
    recorded_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stock_prices (
    id BIGSERIAL PRIMARY KEY,
    stock_id BIGINT REFERENCES stocks(id),
    open_price DECIMAL(15,4),
    high_price DECIMAL(15,4),
    low_price DECIMAL(15,4),
    close_price DECIMAL(15,4),
    volume BIGINT,
    price_date DATE NOT NULL,
    UNIQUE(stock_id, price_date)
);
```

### Service
```java
// PortfolioService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class PortfolioService {

    private final PortfolioRepository portfolioRepository;
    private final TransactionRepository transactionRepository;
    private final PortfolioPositionRepository positionRepository;
    private final StockPriceService stockPriceService;
    private final PriceAlertService alertService;

    @Transactional
    public Transaction addTransaction(Long portfolioId, AddTransactionRequest request) {
        Portfolio portfolio = portfolioRepository.findById(portfolioId)
                .orElseThrow(() -> new ResourceNotFoundException("Portfolio not found"));

        Stock stock = stockRepository.findBySymbol(request.getSymbol())
                .orElseThrow(() -> new ResourceNotFoundException("Stock not found: " + request.getSymbol()));

        BigDecimal totalAmount = request.getPrice()
                .multiply(request.getShares())
                .add(request.getFees());

        Transaction transaction = new Transaction();
        transaction.setPortfolio(portfolio);
        transaction.setStock(stock);
        transaction.setTransactionType(request.getType());
        transaction.setShares(request.getShares());
        transaction.setPrice(request.getPrice());
        transaction.setFees(request.getFees());
        transaction.setTotalAmount(totalAmount);
        transaction.setTransactionDate(request.getDate());
        transaction.setNotes(request.getNotes());

        Transaction saved = transactionRepository.save(transaction);

        // Update portfolio position
        updatePosition(portfolioId, stock.getId(), request.getType(), request.getShares(), request.getPrice());

        return saved;
    }

    private void updatePosition(Long portfolioId, Long stockId, String type,
                                 BigDecimal shares, BigDecimal price) {
        PortfolioPosition position = positionRepository
                .findByPortfolioIdAndStockId(portfolioId, stockId)
                .orElseGet(() -> {
                    PortfolioPosition p = new PortfolioPosition();
                    p.setPortfolio(new Portfolio(portfolioId));
                    p.setStock(new Stock(stockId));
                    return p;
                });

        if ("BUY".equals(type)) {
            BigDecimal newTotalCost = position.getAverageCost().multiply(position.getTotalShares())
                    .add(price.multiply(shares));
            BigDecimal newTotalShares = position.getTotalShares().add(shares);
            position.setAverageCost(newTotalCost.divide(newTotalShares, 4, RoundingMode.HALF_UP));
            position.setTotalShares(newTotalShares);
        } else if ("SELL".equals(type)) {
            BigDecimal realizedPnl = price.subtract(position.getAverageCost()).multiply(shares);
            position.setRealizedPnl(position.getRealizedPnl().add(realizedPnl));
            position.setTotalShares(position.getTotalShares().subtract(shares));
        }

        positionRepository.save(position);
    }

    public PortfolioSummary getPortfolioSummary(Long portfolioId) {
        List<PortfolioPosition> positions = positionRepository.findByPortfolioId(portfolioId);
        BigDecimal totalValue = BigDecimal.ZERO;
        BigDecimal totalCost = BigDecimal.ZERO;
        BigDecimal totalRealizedPnl = BigDecimal.ZERO;

        List<PositionSummary> positionSummaries = new ArrayList<>();

        for (PortfolioPosition position : positions) {
            if (position.getTotalShares().compareTo(BigDecimal.ZERO) == 0) continue;

            BigDecimal currentPrice = stockPriceService.getCurrentPrice(position.getStock().getSymbol());
            BigDecimal marketValue = currentPrice.multiply(position.getTotalShares());
            BigDecimal costBasis = position.getAverageCost().multiply(position.getTotalShares());
            BigDecimal unrealizedPnl = marketValue.subtract(costBasis);
            BigDecimal unrealizedPnlPercent = costBasis.compareTo(BigDecimal.ZERO) != 0
                    ? unrealizedPnl.divide(costBasis, 4, RoundingMode.HALF_UP).multiply(BigDecimal.valueOf(100))
                    : BigDecimal.ZERO;

            positionSummaries.add(new PositionSummary(position, currentPrice, marketValue,
                    unrealizedPnl, unrealizedPnlPercent));

            totalValue = totalValue.add(marketValue);
            totalCost = totalCost.add(costBasis);
            totalRealizedPnl = totalRealizedPnl.add(position.getRealizedPnl());
        }

        BigDecimal totalUnrealizedPnl = totalValue.subtract(totalCost);
        BigDecimal totalPnl = totalUnrealizedPnl.add(totalRealizedPnl);

        return new PortfolioSummary(totalValue, totalCost, totalUnrealizedPnl,
                totalRealizedPnl, totalPnl, positionSummaries);
    }
}
```

### Controller
```java
// StockController.java
@RestController
@RequestMapping("/api/stocks")
@RequiredArgsConstructor
public class StockController {

    private final PortfolioService portfolioService;
    private final StockPriceService stockPriceService;

    @PostMapping("/portfolios/{id}/transactions")
    public ResponseEntity<TransactionDTO> addTransaction(
            @PathVariable Long id,
            @RequestBody @Valid AddTransactionRequest request,
            Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(TransactionDTO.from(portfolioService.addTransaction(id, request)));
    }

    @GetMapping("/portfolios/{id}/summary")
    public ResponseEntity<PortfolioSummaryDTO> getPortfolioSummary(@PathVariable Long id) {
        return ResponseEntity.ok(PortfolioSummaryDTO.from(portfolioService.getPortfolioSummary(id)));
    }

    @GetMapping("/{symbol}/history")
    public ResponseEntity<List<StockPriceDTO>> getPriceHistory(
            @PathVariable String symbol,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        return ResponseEntity.ok(stockPriceService.getHistory(symbol, from, to).stream()
                .map(StockPriceDTO::from).collect(Collectors.toList()));
    }

    @PostMapping("/alerts")
    public ResponseEntity<PriceAlertDTO> createPriceAlert(
            @RequestBody @Valid CreateAlertRequest request, Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(PriceAlertDTO.from(portfolioService.createPriceAlert(request, getCurrentUserId(auth))));
    }

    @GetMapping("/{symbol}/dividends")
    public ResponseEntity<List<DividendDTO>> getDividends(@PathVariable String symbol) {
        return ResponseEntity.ok(portfolioService.getDividendHistory(symbol));
    }
}
```

---

## โปรเจค 45: Crypto Portfolio Tracker

### ภาพรวม
ระบบติดตาม Portfolio คริปโตเคอร์เรนซีที่รองรับ Wallets, Assets, Transactions, ราคาแบบ Real-time จาก CoinGecko API, การประเมินมูลค่า Portfolio, การคำนวณ P&L และการรายงานภาษี

### Flyway Migration
```sql
-- V1__create_crypto_tables.sql
CREATE TABLE crypto_assets (
    id BIGSERIAL PRIMARY KEY,
    coin_id VARCHAR(100) UNIQUE NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    name VARCHAR(100) NOT NULL,
    current_price DECIMAL(20,8),
    market_cap BIGINT,
    price_change_24h DECIMAL(10,4),
    last_updated TIMESTAMP
);

CREATE TABLE wallets (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    wallet_type VARCHAR(20) DEFAULT 'SPOT',
    exchange VARCHAR(50),
    address VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE wallet_holdings (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT REFERENCES wallets(id) ON DELETE CASCADE,
    asset_id BIGINT REFERENCES crypto_assets(id),
    quantity DECIMAL(30,10) DEFAULT 0,
    average_buy_price DECIMAL(20,8) DEFAULT 0,
    UNIQUE(wallet_id, asset_id)
);

CREATE TABLE crypto_transactions (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT REFERENCES wallets(id),
    asset_id BIGINT REFERENCES crypto_assets(id),
    transaction_type VARCHAR(20) NOT NULL,
    quantity DECIMAL(30,10) NOT NULL,
    price_usd DECIMAL(20,8) NOT NULL,
    fee DECIMAL(20,8) DEFAULT 0,
    fee_currency VARCHAR(20),
    total_usd DECIMAL(20,4) NOT NULL,
    exchange VARCHAR(50),
    tx_hash VARCHAR(200),
    notes TEXT,
    transaction_date TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE price_snapshots (
    id BIGSERIAL PRIMARY KEY,
    asset_id BIGINT REFERENCES crypto_assets(id),
    price_usd DECIMAL(20,8) NOT NULL,
    market_cap BIGINT,
    volume_24h BIGINT,
    snapped_at TIMESTAMP DEFAULT NOW()
);
```

### Service
```java
// CryptoPortfolioService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class CryptoPortfolioService {

    private final WalletRepository walletRepository;
    private final WalletHoldingRepository holdingRepository;
    private final CryptoTransactionRepository transactionRepository;
    private final CryptoAssetRepository assetRepository;
    private final CoinGeckoService coinGeckoService;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String PRICE_CACHE_PREFIX = "crypto:price:";
    private static final Duration PRICE_CACHE_TTL = Duration.ofMinutes(5);

    @Transactional
    public CryptoTransaction recordTransaction(Long walletId, RecordTransactionRequest request) {
        Wallet wallet = walletRepository.findById(walletId)
                .orElseThrow(() -> new ResourceNotFoundException("Wallet not found"));

        CryptoAsset asset = assetRepository.findByCoinId(request.getCoinId())
                .orElseThrow(() -> new ResourceNotFoundException("Asset not found"));

        BigDecimal totalUsd = request.getQuantity().multiply(request.getPriceUsd());

        CryptoTransaction transaction = new CryptoTransaction();
        transaction.setWallet(wallet);
        transaction.setAsset(asset);
        transaction.setTransactionType(request.getType());
        transaction.setQuantity(request.getQuantity());
        transaction.setPriceUsd(request.getPriceUsd());
        transaction.setFee(request.getFee() != null ? request.getFee() : BigDecimal.ZERO);
        transaction.setTotalUsd(totalUsd);
        transaction.setExchange(request.getExchange());
        transaction.setTxHash(request.getTxHash());
        transaction.setTransactionDate(request.getTransactionDate());

        CryptoTransaction saved = transactionRepository.save(transaction);

        // Update holding
        updateHolding(walletId, asset.getId(), request.getType(), request.getQuantity(), request.getPriceUsd());

        return saved;
    }

    private void updateHolding(Long walletId, Long assetId, String type,
                                BigDecimal quantity, BigDecimal price) {
        WalletHolding holding = holdingRepository.findByWalletIdAndAssetId(walletId, assetId)
                .orElseGet(() -> {
                    WalletHolding h = new WalletHolding();
                    h.setWallet(new Wallet(walletId));
                    h.setAsset(new CryptoAsset(assetId));
                    return h;
                });

        switch (type.toUpperCase()) {
            case "BUY", "TRANSFER_IN" -> {
                BigDecimal newCost = holding.getAverageBuyPrice().multiply(holding.getQuantity())
                        .add(price.multiply(quantity));
                BigDecimal newQty = holding.getQuantity().add(quantity);
                holding.setAverageBuyPrice(newQty.compareTo(BigDecimal.ZERO) != 0
                        ? newCost.divide(newQty, 10, RoundingMode.HALF_UP)
                        : BigDecimal.ZERO);
                holding.setQuantity(newQty);
            }
            case "SELL", "TRANSFER_OUT" -> {
                holding.setQuantity(holding.getQuantity().subtract(quantity));
            }
        }

        holdingRepository.save(holding);
    }

    public PortfolioValuation getPortfolioValuation(Long userId) {
        List<Wallet> wallets = walletRepository.findByUserId(userId);
        Map<String, BigDecimal> currentPrices = getCurrentPrices(wallets);

        BigDecimal totalValueUsd = BigDecimal.ZERO;
        BigDecimal totalCostBasis = BigDecimal.ZERO;
        List<AssetValuation> assetValuations = new ArrayList<>();

        for (Wallet wallet : wallets) {
            List<WalletHolding> holdings = holdingRepository.findByWalletId(wallet.getId());
            for (WalletHolding holding : holdings) {
                if (holding.getQuantity().compareTo(BigDecimal.ZERO) <= 0) continue;

                BigDecimal currentPrice = currentPrices.getOrDefault(
                        holding.getAsset().getCoinId(), BigDecimal.ZERO);
                BigDecimal marketValue = currentPrice.multiply(holding.getQuantity());
                BigDecimal costBasis = holding.getAverageBuyPrice().multiply(holding.getQuantity());
                BigDecimal unrealizedPnl = marketValue.subtract(costBasis);

                assetValuations.add(new AssetValuation(holding, currentPrice, marketValue,
                        costBasis, unrealizedPnl));
                totalValueUsd = totalValueUsd.add(marketValue);
                totalCostBasis = totalCostBasis.add(costBasis);
            }
        }

        BigDecimal totalPnl = totalValueUsd.subtract(totalCostBasis);
        BigDecimal pnlPercent = totalCostBasis.compareTo(BigDecimal.ZERO) != 0
                ? totalPnl.divide(totalCostBasis, 4, RoundingMode.HALF_UP).multiply(BigDecimal.valueOf(100))
                : BigDecimal.ZERO;

        return new PortfolioValuation(totalValueUsd, totalCostBasis, totalPnl, pnlPercent, assetValuations);
    }

    public TaxReport generateTaxReport(Long userId, int year) {
        List<CryptoTransaction> transactions = transactionRepository
                .findSellTransactionsByUserAndYear(userId, year);

        List<TaxLot> taxLots = new ArrayList<>();
        BigDecimal totalGains = BigDecimal.ZERO;
        BigDecimal totalLosses = BigDecimal.ZERO;

        for (CryptoTransaction sell : transactions) {
            // FIFO cost basis calculation
            List<CryptoTransaction> buys = transactionRepository
                    .findBuyTransactionsBefore(sell.getWallet().getId(),
                            sell.getAsset().getId(), sell.getTransactionDate());

            BigDecimal costBasis = calculateFifoCostBasis(buys, sell.getQuantity(), sell.getPriceUsd());
            BigDecimal proceeds = sell.getTotalUsd().subtract(sell.getFee());
            BigDecimal gain = proceeds.subtract(costBasis);

            TaxLot lot = new TaxLot(sell, costBasis, proceeds, gain);
            taxLots.add(lot);

            if (gain.compareTo(BigDecimal.ZERO) > 0) {
                totalGains = totalGains.add(gain);
            } else {
                totalLosses = totalLosses.add(gain.abs());
            }
        }

        return new TaxReport(year, totalGains, totalLosses, totalGains.subtract(totalLosses), taxLots);
    }

    private Map<String, BigDecimal> getCurrentPrices(List<Wallet> wallets) {
        Set<String> coinIds = wallets.stream()
                .flatMap(w -> holdingRepository.findByWalletId(w.getId()).stream())
                .map(h -> h.getAsset().getCoinId())
                .collect(Collectors.toSet());

        return coinIds.stream().collect(Collectors.toMap(
                id -> id,
                id -> {
                    String cached = (String) redisTemplate.opsForValue().get(PRICE_CACHE_PREFIX + id);
                    if (cached != null) return new BigDecimal(cached);
                    BigDecimal price = coinGeckoService.getPrice(id);
                    redisTemplate.opsForValue().set(PRICE_CACHE_PREFIX + id,
                            price.toString(), PRICE_CACHE_TTL);
                    return price;
                }
        ));
    }

    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void updatePrices() {
        List<CryptoAsset> assets = assetRepository.findAll();
        List<String> coinIds = assets.stream()
                .map(CryptoAsset::getCoinId).collect(Collectors.toList());

        Map<String, BigDecimal> prices = coinGeckoService.getPrices(coinIds);

        prices.forEach((coinId, price) -> {
            assetRepository.updatePrice(coinId, price, LocalDateTime.now());
            redisTemplate.opsForValue().set(PRICE_CACHE_PREFIX + coinId,
                    price.toString(), PRICE_CACHE_TTL);
        });
    }
}
```

### Controller
```java
// CryptoController.java
@RestController
@RequestMapping("/api/crypto")
@RequiredArgsConstructor
public class CryptoController {

    private final CryptoPortfolioService cryptoService;

    @PostMapping("/wallets")
    public ResponseEntity<WalletDTO> createWallet(@RequestBody @Valid CreateWalletRequest request,
                                                   Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(WalletDTO.from(cryptoService.createWallet(request, getCurrentUserId(auth))));
    }

    @PostMapping("/wallets/{id}/transactions")
    public ResponseEntity<TransactionDTO> recordTransaction(
            @PathVariable Long id,
            @RequestBody @Valid RecordTransactionRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(TransactionDTO.from(cryptoService.recordTransaction(id, request)));
    }

    @GetMapping("/portfolio/valuation")
    public ResponseEntity<PortfolioValuationDTO> getPortfolioValuation(Authentication auth) {
        return ResponseEntity.ok(PortfolioValuationDTO.from(
                cryptoService.getPortfolioValuation(getCurrentUserId(auth))));
    }

    @GetMapping("/portfolio/tax-report/{year}")
    public ResponseEntity<TaxReportDTO> getTaxReport(@PathVariable int year, Authentication auth) {
        return ResponseEntity.ok(TaxReportDTO.from(
                cryptoService.generateTaxReport(getCurrentUserId(auth), year)));
    }

    @GetMapping("/assets/{coinId}/price")
    public ResponseEntity<Map<String, Object>> getAssetPrice(@PathVariable String coinId) {
        return ResponseEntity.ok(cryptoService.getAssetPriceInfo(coinId));
    }

    @GetMapping("/assets/search")
    public ResponseEntity<List<CryptoAssetDTO>> searchAssets(@RequestParam String q) {
        return ResponseEntity.ok(cryptoService.searchAssets(q));
    }
}
```

### Docker Compose
```yaml
# docker-compose.yml (Part 109)
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/datadb
      SPRING_REDIS_HOST: redis
      COINGECKO_API_URL: https://api.coingecko.com/api/v3
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: datadb
      POSTGRES_USER: datauser
      POSTGRES_PASSWORD: datapass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  scheduler:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/datadb
      SPRING_REDIS_HOST: redis
      APP_SCHEDULER_ENABLED: "true"
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
  redis_data:
```

---

## สรุป Part 109

| โปรเจค | เทคโนโลยีหลัก | ความซับซ้อน |
|--------|--------------|------------|
| 41. Social Media API | Feed Algorithm, Hashtags, Redis | สูงมาก |
| 42. News Aggregator | RSS Parsing, Personalization | สูง |
| 43. Weather Aggregator | Multi-provider, Cache, Fallback | กลาง |
| 44. Stock Tracker | P&L Calculation, Portfolio | สูง |
| 45. Crypto Portfolio | CoinGecko API, Tax Report, FIFO | สูงมาก |

---

## Navigation

- [← Part 108: Content & Document](part-108-content-document.md)
- [Part 110: Logistics & Tracking →](part-110-logistics-tracking.md)
- [กลับหน้าหลัก](README.md)

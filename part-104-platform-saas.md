# Part 104: โปรเจค 16-20 — Platform & SaaS

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 5 โปรเจคสมบูรณ์

ในส่วนนี้เราจะสร้างแพลตฟอร์มและ SaaS ที่ใช้งานจริง ครอบคลุมระบบบล็อก ฟอรัม Job Board ตลาด Freelance และ LMS

---

## โปรเจคที่ 16: Blog/CMS Platform

### ภาพรวมระบบ

แพลตฟอร์มบล็อกและ CMS ที่รองรับโพสต์ หมวดหมู่ แท็ก ความคิดเห็น ผู้เขียน สถานะ Draft/Publish/Schedule, SEO Metadata, การอัพโหลดรูปภาพ และ RSS Feed

### Entity Classes

```java
// BlogPost.java
@Entity
@Table(name = "blog_posts")
@Data
@NoArgsConstructor
public class BlogPost {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(unique = true, nullable = false)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Column(columnDefinition = "TEXT")
    private String excerpt;

    private String featuredImageUrl;

    @Enumerated(EnumType.STRING)
    private PostStatus status = PostStatus.DRAFT;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private User author;

    @ManyToMany
    @JoinTable(name = "post_categories",
               joinColumns = @JoinColumn(name = "post_id"),
               inverseJoinColumns = @JoinColumn(name = "category_id"))
    private List<BlogCategory> categories = new ArrayList<>();

    @ManyToMany
    @JoinTable(name = "post_tags",
               joinColumns = @JoinColumn(name = "post_id"),
               inverseJoinColumns = @JoinColumn(name = "tag_id"))
    private List<Tag> tags = new ArrayList<>();

    // SEO
    private String metaTitle;
    private String metaDescription;
    private String metaKeywords;

    private Integer viewCount = 0;
    private Integer readTimeMinutes;

    private LocalDateTime publishedAt;
    private LocalDateTime scheduledAt;

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL)
    private List<Comment> comments = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// BlogCategory.java
@Entity
@Table(name = "blog_categories")
@Data
@NoArgsConstructor
public class BlogCategory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(unique = true)
    private String slug;

    private String description;
    private String imageUrl;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private BlogCategory parent;
}

// Tag.java
@Entity
@Table(name = "tags")
@Data
@NoArgsConstructor
public class Tag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String name;

    @Column(unique = true)
    private String slug;

    private String description;
}

// Comment.java
@Entity
@Table(name = "comments")
@Data
@NoArgsConstructor
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    private BlogPost post;

    private String authorName;
    private String authorEmail;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user; // If logged in

    @Column(columnDefinition = "TEXT")
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Comment parent; // For replies

    @OneToMany(mappedBy = "parent")
    private List<Comment> replies = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private CommentStatus status = CommentStatus.PENDING;

    @CreatedDate
    private LocalDateTime createdAt;
}

// MediaFile.java
@Entity
@Table(name = "media_files")
@Data
@NoArgsConstructor
public class MediaFile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String filename;
    private String originalFilename;
    private String mimeType;
    private Long fileSize;
    private String url;
    private String thumbnailUrl;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "uploaded_by")
    private User uploadedBy;

    private String altText;
    private String caption;

    @CreatedDate
    private LocalDateTime uploadedAt;
}

public enum PostStatus { DRAFT, REVIEW, PUBLISHED, SCHEDULED, ARCHIVED }
public enum CommentStatus { PENDING, APPROVED, SPAM, DELETED }
```

### Service Layer

```java
// BlogService.java
@Service
@Transactional
@RequiredArgsConstructor
public class BlogService {

    private final BlogPostRepository postRepository;
    private final BlogCategoryRepository categoryRepository;
    private final TagRepository tagRepository;
    private final CommentRepository commentRepository;
    private final MediaStorageService storageService;

    public BlogPostDTO createPost(CreatePostRequest request, Long authorId) {
        BlogPost post = new BlogPost();
        post.setTitle(request.getTitle());
        post.setSlug(generateSlug(request.getTitle()));
        post.setContent(request.getContent());
        post.setExcerpt(request.getExcerpt() != null ? request.getExcerpt() :
                        generateExcerpt(request.getContent()));
        post.setFeaturedImageUrl(request.getFeaturedImageUrl());
        post.setMetaTitle(request.getMetaTitle() != null ? request.getMetaTitle() : request.getTitle());
        post.setMetaDescription(request.getMetaDescription());
        post.setMetaKeywords(request.getMetaKeywords());
        post.setReadTimeMinutes(calculateReadTime(request.getContent()));

        // Set author
        User author = new User();
        author.setId(authorId);
        post.setAuthor(author);

        // Set categories
        if (request.getCategoryIds() != null) {
            post.setCategories(categoryRepository.findAllById(request.getCategoryIds()));
        }

        // Set or create tags
        if (request.getTags() != null) {
            List<Tag> tags = request.getTags().stream()
                .map(tagName -> tagRepository.findBySlug(slugify(tagName))
                    .orElseGet(() -> {
                        Tag tag = new Tag();
                        tag.setName(tagName);
                        tag.setSlug(slugify(tagName));
                        return tagRepository.save(tag);
                    }))
                .collect(Collectors.toList());
            post.setTags(tags);
        }

        // Handle status
        if (request.getStatus() == PostStatus.PUBLISHED) {
            post.setPublishedAt(LocalDateTime.now());
        } else if (request.getStatus() == PostStatus.SCHEDULED && request.getScheduledAt() != null) {
            post.setScheduledAt(request.getScheduledAt());
        }
        post.setStatus(request.getStatus() != null ? request.getStatus() : PostStatus.DRAFT);

        return toDTO(postRepository.save(post));
    }

    public BlogPostDTO publishPost(Long postId) {
        BlogPost post = postRepository.findById(postId)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));

        if (post.getStatus() == PostStatus.PUBLISHED) {
            throw new BadRequestException("Post is already published");
        }

        post.setStatus(PostStatus.PUBLISHED);
        post.setPublishedAt(LocalDateTime.now());

        return toDTO(postRepository.save(post));
    }

    @Scheduled(cron = "0 * * * * *") // Every minute - check scheduled posts
    public void publishScheduledPosts() {
        List<BlogPost> scheduledPosts = postRepository
            .findByStatusAndScheduledAtBefore(PostStatus.SCHEDULED, LocalDateTime.now());

        for (BlogPost post : scheduledPosts) {
            post.setStatus(PostStatus.PUBLISHED);
            post.setPublishedAt(LocalDateTime.now());
            postRepository.save(post);
        }
    }

    public CommentDTO addComment(Long postId, AddCommentRequest request, Long userId) {
        BlogPost post = postRepository.findById(postId)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));

        if (post.getStatus() != PostStatus.PUBLISHED) {
            throw new BadRequestException("Cannot comment on unpublished posts");
        }

        Comment comment = new Comment();
        comment.setPost(post);
        comment.setContent(request.getContent());

        if (userId != null) {
            User user = new User();
            user.setId(userId);
            comment.setUser(user);
            comment.setStatus(CommentStatus.APPROVED); // Auto-approve logged-in users
        } else {
            comment.setAuthorName(request.getAuthorName());
            comment.setAuthorEmail(request.getAuthorEmail());
            comment.setStatus(CommentStatus.PENDING); // Moderate guests
        }

        if (request.getParentId() != null) {
            Comment parent = commentRepository.findById(request.getParentId())
                .orElseThrow(() -> new ResourceNotFoundException("Parent comment not found"));
            comment.setParent(parent);
        }

        return toCommentDTO(commentRepository.save(comment));
    }

    public Page<BlogPostDTO> getPublishedPosts(String category, String tag,
                                                String search, Pageable pageable) {
        if (search != null && !search.isBlank()) {
            return postRepository.searchPublished(search, pageable).map(this::toDTO);
        }
        if (category != null) {
            return postRepository.findByStatusAndCategorySlug(PostStatus.PUBLISHED, category, pageable).map(this::toDTO);
        }
        if (tag != null) {
            return postRepository.findByStatusAndTagSlug(PostStatus.PUBLISHED, tag, pageable).map(this::toDTO);
        }
        return postRepository.findByStatusOrderByPublishedAtDesc(PostStatus.PUBLISHED, pageable).map(this::toDTO);
    }

    public String generateRssFeed(String baseUrl) {
        List<BlogPost> recentPosts = postRepository
            .findTop20ByStatusOrderByPublishedAtDesc(PostStatus.PUBLISHED);

        StringBuilder rss = new StringBuilder();
        rss.append("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
        rss.append("<rss version=\"2.0\">\n<channel>\n");
        rss.append("<title>Blog Feed</title>\n");
        rss.append("<link>").append(baseUrl).append("</link>\n");
        rss.append("<description>Latest blog posts</description>\n");

        for (BlogPost post : recentPosts) {
            rss.append("<item>\n");
            rss.append("<title>").append(escapeXml(post.getTitle())).append("</title>\n");
            rss.append("<link>").append(baseUrl).append("/posts/").append(post.getSlug()).append("</link>\n");
            rss.append("<description>").append(escapeXml(post.getExcerpt())).append("</description>\n");
            rss.append("<pubDate>").append(post.getPublishedAt()).append("</pubDate>\n");
            rss.append("</item>\n");
        }

        rss.append("</channel>\n</rss>");
        return rss.toString();
    }

    private String generateSlug(String title) {
        String slug = title.toLowerCase()
            .replaceAll("[^a-z0-9\\s-]", "")
            .replaceAll("\\s+", "-")
            .replaceAll("-+", "-");

        if (postRepository.existsBySlug(slug)) {
            slug = slug + "-" + System.currentTimeMillis();
        }
        return slug;
    }

    private String slugify(String text) {
        return text.toLowerCase().replaceAll("[^a-z0-9]", "-").replaceAll("-+", "-");
    }

    private String generateExcerpt(String content) {
        String stripped = content.replaceAll("<[^>]*>", "");
        return stripped.length() > 200 ? stripped.substring(0, 200) + "..." : stripped;
    }

    private int calculateReadTime(String content) {
        int wordCount = content.split("\\s+").length;
        return Math.max(1, wordCount / 200); // 200 words per minute
    }

    private String escapeXml(String text) {
        if (text == null) return "";
        return text.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;");
    }

    private BlogPostDTO toDTO(BlogPost post) {
        return BlogPostDTO.builder()
            .id(post.getId())
            .title(post.getTitle())
            .slug(post.getSlug())
            .excerpt(post.getExcerpt())
            .status(post.getStatus())
            .authorName(post.getAuthor() != null ? post.getAuthor().getUsername() : null)
            .viewCount(post.getViewCount())
            .readTimeMinutes(post.getReadTimeMinutes())
            .publishedAt(post.getPublishedAt())
            .categories(post.getCategories().stream().map(BlogCategory::getName).collect(Collectors.toList()))
            .tags(post.getTags().stream().map(Tag::getName).collect(Collectors.toList()))
            .build();
    }

    private CommentDTO toCommentDTO(Comment c) {
        return CommentDTO.builder()
            .id(c.getId())
            .authorName(c.getUser() != null ? c.getUser().getUsername() : c.getAuthorName())
            .content(c.getContent())
            .status(c.getStatus())
            .createdAt(c.getCreatedAt())
            .build();
    }
}
```

### REST Controller

```java
// BlogController.java
@RestController
@RequestMapping("/api/v1/blog")
@RequiredArgsConstructor
public class BlogController {

    private final BlogService blogService;

    @GetMapping("/posts")
    public ResponseEntity<Page<BlogPostDTO>> getPosts(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String tag,
            @RequestParam(required = false) String search,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("publishedAt").descending());
        return ResponseEntity.ok(blogService.getPublishedPosts(category, tag, search, pageable));
    }

    @GetMapping("/posts/{slug}")
    public ResponseEntity<BlogPostDTO> getPost(@PathVariable String slug) {
        BlogPostDTO post = blogService.getPostBySlug(slug);
        blogService.incrementViewCount(post.getId());
        return ResponseEntity.ok(post);
    }

    @PostMapping("/posts")
    @PreAuthorize("hasRole('AUTHOR') or hasRole('ADMIN')")
    public ResponseEntity<BlogPostDTO> createPost(
            @Valid @RequestBody CreatePostRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.status(HttpStatus.CREATED).body(blogService.createPost(request, user.getId()));
    }

    @PutMapping("/posts/{id}")
    @PreAuthorize("hasRole('AUTHOR') or hasRole('ADMIN')")
    public ResponseEntity<BlogPostDTO> updatePost(
            @PathVariable Long id,
            @Valid @RequestBody UpdatePostRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(blogService.updatePost(id, request, user.getId()));
    }

    @PostMapping("/posts/{id}/publish")
    @PreAuthorize("hasRole('EDITOR') or hasRole('ADMIN')")
    public ResponseEntity<BlogPostDTO> publishPost(@PathVariable Long id) {
        return ResponseEntity.ok(blogService.publishPost(id));
    }

    @PostMapping("/posts/{id}/comments")
    public ResponseEntity<CommentDTO> addComment(
            @PathVariable Long id,
            @Valid @RequestBody AddCommentRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        Long userId = user != null ? user.getId() : null;
        return ResponseEntity.status(HttpStatus.CREATED).body(blogService.addComment(id, request, userId));
    }

    @PostMapping("/media/upload")
    @PreAuthorize("hasRole('AUTHOR') or hasRole('ADMIN')")
    public ResponseEntity<MediaFileDTO> uploadMedia(
            @RequestParam("file") MultipartFile file,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(blogService.uploadMedia(file, user.getId()));
    }

    @GetMapping("/rss")
    public ResponseEntity<String> getRssFeed(HttpServletRequest request) {
        String baseUrl = request.getScheme() + "://" + request.getServerName();
        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_RSS_XML)
            .body(blogService.generateRssFeed(baseUrl));
    }
}
```

---

## โปรเจคที่ 17: Forum/Community Platform

### ภาพรวมระบบ

แพลตฟอร์มฟอรัมและชุมชนที่รองรับหมวดหมู่ กระทู้ โพสต์ การโหวต (upvote/downvote) ป้าย (badges) ชื่อเสียงผู้ใช้ การจัดการเนื้อหา และการค้นหา

### Entity Classes

```java
// ForumCategory.java
@Entity
@Table(name = "forum_categories")
@Data
@NoArgsConstructor
public class ForumCategory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String slug;
    private String description;
    private String icon;
    private Integer displayOrder;
    private Boolean moderated = false;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private ForumCategory parent;

    private Integer threadCount = 0;
    private Integer postCount = 0;
}

// Thread.java
@Entity
@Table(name = "threads")
@Data
@NoArgsConstructor
public class Thread {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private ForumCategory category;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private User author;

    @Enumerated(EnumType.STRING)
    private ThreadStatus status = ThreadStatus.OPEN;

    private Boolean pinned = false;
    private Boolean locked = false;

    private Integer viewCount = 0;
    private Integer replyCount = 0;
    private Integer upvotes = 0;
    private Integer downvotes = 0;

    @OneToMany(mappedBy = "thread", cascade = CascadeType.ALL)
    @OrderBy("createdAt ASC")
    private List<ForumPost> posts = new ArrayList<>();

    private LocalDateTime lastActivityAt;

    @CreatedDate
    private LocalDateTime createdAt;
}

// ForumPost.java
@Entity
@Table(name = "forum_posts")
@Data
@NoArgsConstructor
public class ForumPost {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "thread_id")
    private Thread thread;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private User author;

    @Column(columnDefinition = "TEXT")
    private String content;

    private Integer upvotes = 0;
    private Integer downvotes = 0;

    private Boolean isAnswer = false; // Marked as accepted answer

    @Enumerated(EnumType.STRING)
    private PostStatus status = PostStatus.VISIBLE;

    private String editReason;
    private LocalDateTime editedAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private ForumPost parent; // For quotes/replies

    @CreatedDate
    private LocalDateTime createdAt;
}

// Vote.java
@Entity
@Table(name = "votes",
       uniqueConstraints = @UniqueConstraint(columnNames = {"user_id", "target_type", "target_id"}))
@Data
@NoArgsConstructor
public class Vote {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    private String targetType; // THREAD or POST
    private Long targetId;

    private Integer value; // 1 for upvote, -1 for downvote
}

// Badge.java
@Entity
@Table(name = "badges")
@Data
@NoArgsConstructor
public class Badge {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private String icon;
    private String type; // GOLD, SILVER, BRONZE
    private String condition; // JSON condition
}

// UserReputation.java
@Entity
@Table(name = "user_reputations")
@Data
@NoArgsConstructor
public class UserReputation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true)
    private User user;

    private Integer score = 0;
    private Integer threadCount = 0;
    private Integer postCount = 0;
    private Integer helpfulAnswers = 0;
}

public enum ThreadStatus { OPEN, CLOSED, ARCHIVED }
```

### Service Layer

```java
// ForumService.java
@Service
@Transactional
@RequiredArgsConstructor
public class ForumService {

    private final ThreadRepository threadRepository;
    private final ForumPostRepository postRepository;
    private final VoteRepository voteRepository;
    private final UserReputationRepository reputationRepository;
    private final BadgeRepository badgeRepository;

    public ThreadDTO createThread(Long categoryId, Long authorId, CreateThreadRequest request) {
        Thread thread = new Thread();
        thread.setTitle(request.getTitle());
        thread.setCategory(new ForumCategory()); // Set from repo
        thread.setAuthor(new User()); // Set from repo

        // First post is the content
        ForumPost firstPost = new ForumPost();
        firstPost.setThread(thread);
        firstPost.setAuthor(thread.getAuthor());
        firstPost.setContent(request.getContent());
        thread.getPosts().add(firstPost);

        thread.setLastActivityAt(LocalDateTime.now());

        Thread saved = threadRepository.save(thread);

        // Update category counters
        updateCategoryCounters(categoryId);

        // Update reputation
        addReputation(authorId, 5, "Created thread");

        return toThreadDTO(saved);
    }

    public ForumPostDTO replyToThread(Long threadId, Long authorId, String content) {
        Thread thread = threadRepository.findById(threadId)
            .orElseThrow(() -> new ResourceNotFoundException("Thread not found"));

        if (thread.isLocked()) {
            throw new BadRequestException("Thread is locked");
        }

        ForumPost post = new ForumPost();
        post.setThread(thread);
        User author = new User();
        author.setId(authorId);
        post.setAuthor(author);
        post.setContent(content);

        thread.setReplyCount(thread.getReplyCount() + 1);
        thread.setLastActivityAt(LocalDateTime.now());
        threadRepository.save(thread);

        ForumPost saved = postRepository.save(post);

        // Update reputation
        addReputation(authorId, 2, "Posted reply");

        return toPostDTO(saved);
    }

    public VoteResult vote(Long userId, String targetType, Long targetId, int voteValue) {
        // Check for existing vote
        Optional<Vote> existing = voteRepository.findByUserIdAndTargetTypeAndTargetId(userId, targetType, targetId);

        if (existing.isPresent()) {
            Vote existingVote = existing.get();
            if (existingVote.getValue() == voteValue) {
                // Remove vote (toggle off)
                voteRepository.delete(existingVote);
                updateVoteCount(targetType, targetId, -voteValue);
                return new VoteResult("removed", getVoteCount(targetType, targetId));
            } else {
                // Change vote
                int change = voteValue - existingVote.getValue();
                existingVote.setValue(voteValue);
                voteRepository.save(existingVote);
                updateVoteCount(targetType, targetId, change);
                return new VoteResult("changed", getVoteCount(targetType, targetId));
            }
        }

        Vote vote = new Vote();
        vote.setUser(new User());
        vote.setTargetType(targetType);
        vote.setTargetId(targetId);
        vote.setValue(voteValue);
        voteRepository.save(vote);

        updateVoteCount(targetType, targetId, voteValue);

        // Award reputation to post author
        if ("POST".equals(targetType) && voteValue > 0) {
            ForumPost post = postRepository.findById(targetId).orElse(null);
            if (post != null) {
                addReputation(post.getAuthor().getId(), 10, "Upvoted");
            }
        }

        return new VoteResult("added", getVoteCount(targetType, targetId));
    }

    public ForumPostDTO markAsAnswer(Long postId, Long userId) {
        ForumPost post = postRepository.findById(postId)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));

        Thread thread = post.getThread();

        // Only thread author can mark answer
        if (!thread.getAuthor().getId().equals(userId)) {
            throw new ForbiddenException("Only thread author can mark accepted answer");
        }

        // Remove previous answer
        postRepository.clearAnswers(thread.getId());

        post.setIsAnswer(true);
        postRepository.save(post);

        // Award reputation to answerer
        addReputation(post.getAuthor().getId(), 15, "Answer accepted");

        return toPostDTO(post);
    }

    public Page<ThreadDTO> searchThreads(String query, Long categoryId, Pageable pageable) {
        if (categoryId != null) {
            return threadRepository.searchByCategory(query, categoryId, pageable).map(this::toThreadDTO);
        }
        return threadRepository.search(query, pageable).map(this::toThreadDTO);
    }

    private void updateVoteCount(String targetType, Long targetId, int change) {
        if ("THREAD".equals(targetType)) {
            Thread thread = threadRepository.findById(targetId).orElse(null);
            if (thread != null) {
                if (change > 0) thread.setUpvotes(thread.getUpvotes() + change);
                else thread.setDownvotes(thread.getDownvotes() - change);
                threadRepository.save(thread);
            }
        } else if ("POST".equals(targetType)) {
            ForumPost post = postRepository.findById(targetId).orElse(null);
            if (post != null) {
                if (change > 0) post.setUpvotes(post.getUpvotes() + change);
                else post.setDownvotes(post.getDownvotes() - change);
                postRepository.save(post);
            }
        }
    }

    private int getVoteCount(String targetType, Long targetId) {
        if ("THREAD".equals(targetType)) {
            Thread thread = threadRepository.findById(targetId).orElse(null);
            return thread != null ? thread.getUpvotes() - thread.getDownvotes() : 0;
        }
        ForumPost post = postRepository.findById(targetId).orElse(null);
        return post != null ? post.getUpvotes() - post.getDownvotes() : 0;
    }

    private void addReputation(Long userId, int points, String reason) {
        UserReputation rep = reputationRepository.findByUserId(userId)
            .orElseGet(() -> {
                UserReputation r = new UserReputation();
                r.setUser(new User()); // Load from repo
                return r;
            });
        rep.setScore(rep.getScore() + points);
        reputationRepository.save(rep);

        checkAndAwardBadges(userId, rep.getScore());
    }

    private void checkAndAwardBadges(Long userId, int reputation) {
        if (reputation >= 100) awardBadgeIfNotExists(userId, "BRONZE_CONTRIBUTOR");
        if (reputation >= 500) awardBadgeIfNotExists(userId, "SILVER_CONTRIBUTOR");
        if (reputation >= 1000) awardBadgeIfNotExists(userId, "GOLD_CONTRIBUTOR");
    }

    private void awardBadgeIfNotExists(Long userId, String badgeName) {
        // Check and award badge logic
    }

    private void updateCategoryCounters(Long categoryId) {
        // Update thread/post counts in category
    }

    private ThreadDTO toThreadDTO(Thread t) {
        return ThreadDTO.builder()
            .id(t.getId())
            .title(t.getTitle())
            .authorName(t.getAuthor().getUsername())
            .status(t.getStatus())
            .viewCount(t.getViewCount())
            .replyCount(t.getReplyCount())
            .upvotes(t.getUpvotes())
            .pinned(t.getPinned())
            .lastActivityAt(t.getLastActivityAt())
            .createdAt(t.getCreatedAt())
            .build();
    }

    private ForumPostDTO toPostDTO(ForumPost p) {
        return ForumPostDTO.builder()
            .id(p.getId())
            .content(p.getContent())
            .authorName(p.getAuthor().getUsername())
            .upvotes(p.getUpvotes())
            .downvotes(p.getDownvotes())
            .isAnswer(p.getIsAnswer())
            .createdAt(p.getCreatedAt())
            .build();
    }
}
```

---

## โปรเจคที่ 18: Job Board Platform

### ภาพรวมระบบ

แพลตฟอร์ม Job Board ที่รองรับประกาศรับสมัครงาน ข้อมูลบริษัท การสมัครงาน การอัพโหลด Resume การแจ้งเตือนงาน การติดตามสถานะ และ Dashboard สำหรับ Recruiter

### Entity Classes

```java
// JobListing.java
@Entity
@Table(name = "job_listings")
@Data
@NoArgsConstructor
public class JobListing {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "company_id")
    private JobCompany company;

    private String location;
    private Boolean remote = false;

    private String employmentType; // FULL_TIME, PART_TIME, CONTRACT, INTERNSHIP
    private String experienceLevel; // ENTRY, MID, SENIOR, LEAD, DIRECTOR

    @Column(precision = 12, scale = 2)
    private BigDecimal salaryMin;

    @Column(precision = 12, scale = 2)
    private BigDecimal salaryMax;

    private String salaryCurrency = "THB";
    private Boolean salaryVisible = true;

    @ElementCollection
    @CollectionTable(name = "job_skills")
    private List<String> requiredSkills = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "job_benefits")
    private List<String> benefits = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private JobStatus status = JobStatus.DRAFT;

    private LocalDate applicationDeadline;
    private Integer maxApplications;
    private Integer applicationCount = 0;
    private Integer viewCount = 0;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "posted_by")
    private User postedBy;

    private LocalDateTime publishedAt;
    private LocalDateTime expiresAt;

    @CreatedDate
    private LocalDateTime createdAt;
}

// JobCompany.java
@Entity
@Table(name = "job_companies")
@Data
@NoArgsConstructor
public class JobCompany {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private String industry;
    private String website;
    private String logoUrl;
    private String location;
    private String companySize; // STARTUP, SME, LARGE, ENTERPRISE
    private String foundedYear;
    private Boolean verified = false;
}

// JobApplication.java
@Entity
@Table(name = "job_applications",
       uniqueConstraints = @UniqueConstraint(columnNames = {"job_listing_id", "applicant_id"}))
@Data
@NoArgsConstructor
public class JobApplication {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "job_listing_id")
    private JobListing jobListing;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "applicant_id")
    private User applicant;

    @Column(columnDefinition = "TEXT")
    private String coverLetter;

    private String resumeUrl;
    private String portfolioUrl;

    @Enumerated(EnumType.STRING)
    private ApplicationStatus status = ApplicationStatus.SUBMITTED;

    private String rejectionReason;
    private String recruiterNotes;

    private LocalDateTime reviewedAt;
    private LocalDateTime interviewScheduledAt;
    private LocalDateTime interviewCompletedAt;

    @CreatedDate
    private LocalDateTime appliedAt;
}

// JobAlert.java
@Entity
@Table(name = "job_alerts")
@Data
@NoArgsConstructor
public class JobAlert {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    private String keyword;
    private String location;
    private String employmentType;
    private String experienceLevel;

    @Column(precision = 12, scale = 2)
    private BigDecimal minSalary;

    private Boolean remote;
    private Boolean active = true;

    @Enumerated(EnumType.STRING)
    private AlertFrequency frequency = AlertFrequency.DAILY;

    private LocalDateTime lastSentAt;
}

public enum JobStatus { DRAFT, PUBLISHED, PAUSED, CLOSED, EXPIRED }
public enum ApplicationStatus { SUBMITTED, VIEWED, SHORTLISTED, INTERVIEW, OFFER, ACCEPTED, REJECTED, WITHDRAWN }
public enum AlertFrequency { IMMEDIATE, DAILY, WEEKLY }
```

### Service Layer

```java
// JobBoardService.java
@Service
@Transactional
@RequiredArgsConstructor
public class JobBoardService {

    private final JobListingRepository jobRepository;
    private final JobApplicationRepository applicationRepository;
    private final JobAlertRepository alertRepository;
    private final EmailService emailService;

    public Page<JobListingDTO> searchJobs(JobSearchRequest request, Pageable pageable) {
        Specification<JobListing> spec = Specification.where(
            JobSpecification.hasStatus(JobStatus.PUBLISHED)
                .and(JobSpecification.keywordMatch(request.getKeyword()))
                .and(JobSpecification.locationMatch(request.getLocation()))
                .and(JobSpecification.salaryRange(request.getMinSalary(), request.getMaxSalary()))
                .and(JobSpecification.employmentType(request.getEmploymentType()))
                .and(JobSpecification.experienceLevel(request.getExperienceLevel()))
                .and(JobSpecification.isRemote(request.getRemote()))
        );

        return jobRepository.findAll(spec, pageable).map(this::toDTO);
    }

    public ApplicationDTO applyForJob(Long jobId, Long applicantId, ApplyJobRequest request) {
        JobListing job = jobRepository.findById(jobId)
            .orElseThrow(() -> new ResourceNotFoundException("Job not found"));

        if (job.getStatus() != JobStatus.PUBLISHED) {
            throw new BadRequestException("Job is not accepting applications");
        }

        if (job.getApplicationDeadline() != null && 
            LocalDate.now().isAfter(job.getApplicationDeadline())) {
            throw new BadRequestException("Application deadline has passed");
        }

        if (applicationRepository.existsByJobListingIdAndApplicantId(jobId, applicantId)) {
            throw new ConflictException("You have already applied for this job");
        }

        if (job.getMaxApplications() != null && 
            job.getApplicationCount() >= job.getMaxApplications()) {
            throw new ConflictException("Job has reached maximum applications");
        }

        JobApplication application = new JobApplication();
        application.setJobListing(job);
        User applicant = new User();
        applicant.setId(applicantId);
        application.setApplicant(applicant);
        application.setCoverLetter(request.getCoverLetter());
        application.setResumeUrl(request.getResumeUrl());
        application.setPortfolioUrl(request.getPortfolioUrl());

        job.setApplicationCount(job.getApplicationCount() + 1);
        jobRepository.save(job);

        JobApplication saved = applicationRepository.save(application);

        // Notify recruiter
        emailService.notifyNewApplication(job.getPostedBy().getEmail(), saved);

        return toApplicationDTO(saved);
    }

    public ApplicationDTO updateApplicationStatus(Long applicationId, ApplicationStatus status, 
                                                    String notes, Long recruiterId) {
        JobApplication application = applicationRepository.findById(applicationId)
            .orElseThrow(() -> new ResourceNotFoundException("Application not found"));

        ApplicationStatus oldStatus = application.getStatus();
        application.setStatus(status);
        application.setRecruiterNotes(notes);

        if (status == ApplicationStatus.VIEWED) {
            application.setReviewedAt(LocalDateTime.now());
        } else if (status == ApplicationStatus.REJECTED) {
            application.setRejectionReason(notes);
        }

        // Send email notification to applicant
        emailService.sendApplicationStatusUpdate(
            application.getApplicant().getEmail(), application, oldStatus, status);

        return toApplicationDTO(applicationRepository.save(application));
    }

    public RecruiterDashboard getRecruiterDashboard(Long recruiterId) {
        List<JobListing> jobs = jobRepository.findByPostedById(recruiterId);

        Map<JobStatus, Long> jobsByStatus = jobs.stream()
            .collect(Collectors.groupingBy(JobListing::getStatus, Collectors.counting()));

        List<ApplicationDTO> recentApplications = applicationRepository
            .findRecentByRecruiter(recruiterId, PageRequest.of(0, 10))
            .stream().map(this::toApplicationDTO).collect(Collectors.toList());

        long totalApplications = jobs.stream()
            .mapToLong(JobListing::getApplicationCount).sum();

        return RecruiterDashboard.builder()
            .totalJobs(jobs.size())
            .publishedJobs(jobsByStatus.getOrDefault(JobStatus.PUBLISHED, 0L))
            .totalApplications(totalApplications)
            .pendingReview(applicationRepository.countByStatusAndRecruiter(
                ApplicationStatus.SUBMITTED, recruiterId))
            .recentApplications(recentApplications)
            .build();
    }

    @Scheduled(cron = "0 0 8 * * *") // Daily at 8am
    public void sendJobAlerts() {
        List<JobAlert> activeAlerts = alertRepository
            .findByActiveAndFrequency(true, AlertFrequency.DAILY);

        for (JobAlert alert : activeAlerts) {
            try {
                List<JobListing> matchingJobs = jobRepository.findMatchingJobs(
                    alert.getKeyword(), alert.getLocation(), alert.getEmploymentType(),
                    alert.getMinSalary(), alert.getLastSentAt());

                if (!matchingJobs.isEmpty()) {
                    emailService.sendJobAlert(alert.getUser().getEmail(), alert, matchingJobs);
                    alert.setLastSentAt(LocalDateTime.now());
                    alertRepository.save(alert);
                }
            } catch (Exception e) {
                // Log error but continue
            }
        }
    }

    private JobListingDTO toDTO(JobListing job) {
        return JobListingDTO.builder()
            .id(job.getId())
            .title(job.getTitle())
            .companyName(job.getCompany().getName())
            .location(job.getLocation())
            .remote(job.getRemote())
            .employmentType(job.getEmploymentType())
            .salaryMin(job.getSalaryVisible() ? job.getSalaryMin() : null)
            .salaryMax(job.getSalaryVisible() ? job.getSalaryMax() : null)
            .requiredSkills(job.getRequiredSkills())
            .applicationDeadline(job.getApplicationDeadline())
            .publishedAt(job.getPublishedAt())
            .applicationCount(job.getApplicationCount())
            .build();
    }

    private ApplicationDTO toApplicationDTO(JobApplication app) {
        return ApplicationDTO.builder()
            .id(app.getId())
            .jobTitle(app.getJobListing().getTitle())
            .companyName(app.getJobListing().getCompany().getName())
            .status(app.getStatus())
            .appliedAt(app.getAppliedAt())
            .build();
    }
}
```

---

## โปรเจคที่ 19: Freelance Marketplace

### ภาพรวมระบบ

แพลตฟอร์ม Freelance ที่รองรับการสร้างบริการ (Gigs) การยื่น Proposal สัญญา Milestones การชำระเงินแบบ Escrow รีวิว และระบบจัดการข้อพิพาท

### Entity Classes

```java
// Gig.java
@Entity
@Table(name = "gigs")
@Data
@NoArgsConstructor
public class Gig {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "freelancer_id")
    private User freelancer;

    private String category;
    private String subcategory;

    @Column(precision = 10, scale = 2)
    private BigDecimal startingPrice;

    private Integer deliveryDays;
    private Integer revisions;

    @ElementCollection
    @CollectionTable(name = "gig_tags")
    private List<String> tags = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "gig_images")
    private List<String> imageUrls = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private GigStatus status = GigStatus.ACTIVE;

    private Integer totalOrders = 0;
    private Integer completedOrders = 0;
    private Double averageRating = 0.0;
    private Integer reviewCount = 0;

    @CreatedDate
    private LocalDateTime createdAt;
}

// Contract.java
@Entity
@Table(name = "contracts")
@Data
@NoArgsConstructor
public class Contract {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String contractNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "gig_id")
    private Gig gig;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "client_id")
    private User client;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "freelancer_id")
    private User freelancer;

    @Column(columnDefinition = "TEXT")
    private String requirements;

    @Column(precision = 10, scale = 2)
    private BigDecimal totalAmount;

    private Integer deliveryDays;
    private LocalDate expectedDelivery;
    private LocalDate deliveredAt;

    @Enumerated(EnumType.STRING)
    private ContractStatus status = ContractStatus.ACTIVE;

    @OneToMany(mappedBy = "contract", cascade = CascadeType.ALL)
    private List<Milestone> milestones = new ArrayList<>();

    @OneToMany(mappedBy = "contract")
    private List<Dispute> disputes = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// Milestone.java
@Entity
@Table(name = "milestones")
@Data
@NoArgsConstructor
public class Milestone {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "contract_id")
    private Contract contract;

    private String title;
    private String description;

    @Column(precision = 10, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private MilestoneStatus status = MilestoneStatus.PENDING;

    private LocalDate dueDate;
    private LocalDate completedDate;

    private String deliverableUrl; // Link to deliverables
}

// EscrowAccount.java
@Entity
@Table(name = "escrow_accounts")
@Data
@NoArgsConstructor
public class EscrowAccount {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "contract_id")
    private Contract contract;

    @Column(precision = 12, scale = 2)
    private BigDecimal heldAmount;

    @Column(precision = 12, scale = 2)
    private BigDecimal releasedAmount = BigDecimal.ZERO;

    @Enumerated(EnumType.STRING)
    private EscrowStatus status = EscrowStatus.HOLDING;

    @Column(precision = 5, scale = 2)
    private BigDecimal platformFeePercent = new BigDecimal("10.00");
}

// FreelanceReview.java
@Entity
@Table(name = "freelance_reviews")
@Data
@NoArgsConstructor
public class FreelanceReview {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "contract_id")
    private Contract contract;

    @ManyToOne
    @JoinColumn(name = "reviewer_id")
    private User reviewer;

    @ManyToOne
    @JoinColumn(name = "reviewed_id")
    private User reviewed;

    private Integer rating; // 1-5

    @Column(columnDefinition = "TEXT")
    private String comment;

    private String reviewType; // CLIENT_TO_FREELANCER or FREELANCER_TO_CLIENT

    @CreatedDate
    private LocalDateTime createdAt;
}

// Dispute.java
@Entity
@Table(name = "disputes")
@Data
@NoArgsConstructor
public class Dispute {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "contract_id")
    private Contract contract;

    @ManyToOne
    @JoinColumn(name = "raised_by")
    private User raisedBy;

    private String reason;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Enumerated(EnumType.STRING)
    private DisputeStatus status = DisputeStatus.OPEN;

    private String resolution;
    private LocalDateTime resolvedAt;
}

public enum GigStatus { ACTIVE, PAUSED, DELETED }
public enum ContractStatus { ACTIVE, DELIVERED, COMPLETED, CANCELLED, DISPUTED }
public enum MilestoneStatus { PENDING, IN_PROGRESS, SUBMITTED, APPROVED, REJECTED }
public enum EscrowStatus { HOLDING, PARTIALLY_RELEASED, FULLY_RELEASED, REFUNDED, DISPUTED }
public enum DisputeStatus { OPEN, IN_REVIEW, RESOLVED, CLOSED }
```

### Service Layer

```java
// FreelanceService.java
@Service
@Transactional
@RequiredArgsConstructor
public class FreelanceService {

    private final ContractRepository contractRepository;
    private final GigRepository gigRepository;
    private final MilestoneRepository milestoneRepository;
    private final EscrowAccountRepository escrowRepository;
    private final FreelanceReviewRepository reviewRepository;

    public ContractDTO createContract(Long clientId, CreateContractRequest request) {
        Gig gig = gigRepository.findById(request.getGigId())
            .orElseThrow(() -> new ResourceNotFoundException("Gig not found"));

        if (gig.getStatus() != GigStatus.ACTIVE) {
            throw new BadRequestException("Gig is not active");
        }

        Contract contract = new Contract();
        contract.setContractNumber("CTR-" + System.currentTimeMillis());
        contract.setGig(gig);
        contract.setClient(new User());
        contract.setFreelancer(gig.getFreelancer());
        contract.setRequirements(request.getRequirements());
        contract.setTotalAmount(request.getAgreedAmount());
        contract.setDeliveryDays(request.getDeliveryDays());
        contract.setExpectedDelivery(LocalDate.now().plusDays(request.getDeliveryDays()));

        Contract saved = contractRepository.save(contract);

        // Create escrow account
        EscrowAccount escrow = new EscrowAccount();
        escrow.setContract(saved);
        escrow.setHeldAmount(request.getAgreedAmount());
        escrowRepository.save(escrow);

        // Update gig counter
        gig.setTotalOrders(gig.getTotalOrders() + 1);
        gigRepository.save(gig);

        return toContractDTO(saved);
    }

    public MilestoneDTO submitMilestone(Long contractId, Long milestoneId, 
                                         Long freelancerId, String deliverableUrl) {
        Milestone milestone = milestoneRepository.findByIdAndContractId(milestoneId, contractId)
            .orElseThrow(() -> new ResourceNotFoundException("Milestone not found"));

        Contract contract = milestone.getContract();

        if (!contract.getFreelancer().getId().equals(freelancerId)) {
            throw new ForbiddenException("Only the freelancer can submit milestones");
        }

        milestone.setStatus(MilestoneStatus.SUBMITTED);
        milestone.setDeliverableUrl(deliverableUrl);

        return toMilestoneDTO(milestoneRepository.save(milestone));
    }

    public MilestoneDTO approveMilestone(Long contractId, Long milestoneId, Long clientId) {
        Milestone milestone = milestoneRepository.findByIdAndContractId(milestoneId, contractId)
            .orElseThrow(() -> new ResourceNotFoundException("Milestone not found"));

        Contract contract = milestone.getContract();

        if (!contract.getClient().getId().equals(clientId)) {
            throw new ForbiddenException("Only the client can approve milestones");
        }

        if (milestone.getStatus() != MilestoneStatus.SUBMITTED) {
            throw new BadRequestException("Milestone must be submitted before approval");
        }

        milestone.setStatus(MilestoneStatus.APPROVED);
        milestone.setCompletedDate(LocalDate.now());

        // Release payment from escrow
        releaseEscrowPayment(contract.getId(), milestone.getAmount());

        return toMilestoneDTO(milestoneRepository.save(milestone));
    }

    private void releaseEscrowPayment(Long contractId, BigDecimal amount) {
        EscrowAccount escrow = escrowRepository.findByContractId(contractId)
            .orElseThrow(() -> new ResourceNotFoundException("Escrow account not found"));

        BigDecimal platformFee = amount.multiply(escrow.getPlatformFeePercent())
            .divide(new BigDecimal("100"), 2, RoundingMode.HALF_UP);
        BigDecimal freelancerAmount = amount.subtract(platformFee);

        escrow.setReleasedAmount(escrow.getReleasedAmount().add(amount));

        if (escrow.getReleasedAmount().compareTo(escrow.getHeldAmount()) >= 0) {
            escrow.setStatus(EscrowStatus.FULLY_RELEASED);
        } else {
            escrow.setStatus(EscrowStatus.PARTIALLY_RELEASED);
        }

        escrowRepository.save(escrow);

        // Transfer to freelancer wallet (integrate with WalletService)
        // walletService.credit(contract.getFreelancer().getId(), freelancerAmount, "Milestone payment");
    }

    public ReviewDTO submitReview(Long contractId, Long reviewerId, SubmitReviewRequest request) {
        Contract contract = contractRepository.findById(contractId)
            .orElseThrow(() -> new ResourceNotFoundException("Contract not found"));

        if (contract.getStatus() != ContractStatus.COMPLETED) {
            throw new BadRequestException("Can only review completed contracts");
        }

        boolean isClient = contract.getClient().getId().equals(reviewerId);
        boolean isFreelancer = contract.getFreelancer().getId().equals(reviewerId);

        if (!isClient && !isFreelancer) {
            throw new ForbiddenException("Only contract parties can leave reviews");
        }

        FreelanceReview review = new FreelanceReview();
        review.setContract(contract);
        review.setReviewer(new User());
        review.setReviewed(isClient ? contract.getFreelancer() : contract.getClient());
        review.setRating(request.getRating());
        review.setComment(request.getComment());
        review.setReviewType(isClient ? "CLIENT_TO_FREELANCER" : "FREELANCER_TO_CLIENT");

        reviewRepository.save(review);

        // Update gig rating
        if (isClient) {
            updateGigRating(contract.getGig().getId());
        }

        return ReviewDTO.builder()
            .rating(review.getRating())
            .comment(review.getComment())
            .createdAt(review.getCreatedAt())
            .build();
    }

    private void updateGigRating(Long gigId) {
        Double avg = reviewRepository.getAverageRatingForGig(gigId);
        Long count = reviewRepository.countByGigId(gigId);

        Gig gig = gigRepository.findById(gigId).orElse(null);
        if (gig != null) {
            gig.setAverageRating(avg != null ? avg : 0.0);
            gig.setReviewCount(count.intValue());
            gigRepository.save(gig);
        }
    }

    private ContractDTO toContractDTO(Contract c) {
        return ContractDTO.builder()
            .id(c.getId())
            .contractNumber(c.getContractNumber())
            .gigTitle(c.getGig().getTitle())
            .freelancerName(c.getFreelancer().getUsername())
            .status(c.getStatus())
            .totalAmount(c.getTotalAmount())
            .expectedDelivery(c.getExpectedDelivery())
            .build();
    }

    private MilestoneDTO toMilestoneDTO(Milestone m) {
        return MilestoneDTO.builder()
            .id(m.getId())
            .title(m.getTitle())
            .amount(m.getAmount())
            .status(m.getStatus())
            .dueDate(m.getDueDate())
            .deliverableUrl(m.getDeliverableUrl())
            .build();
    }
}
```

---

## โปรเจคที่ 20: Learning Management System (LMS)

### ภาพรวมระบบ

ระบบ LMS ที่รองรับการสร้างคอร์ส บทเรียน แบบทดสอบ การติดตามความคืบหน้า ใบรับรอง การลงทะเบียน และการจัดการ Instructor

### Entity Classes

```java
// Course.java
@Entity
@Table(name = "courses")
@Data
@NoArgsConstructor
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(columnDefinition = "TEXT")
    private String shortDescription;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "instructor_id")
    private User instructor;

    private String category;
    private String level; // BEGINNER, INTERMEDIATE, ADVANCED
    private String language;
    private String thumbnailUrl;
    private String previewVideoUrl;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private Boolean isFree = false;

    @Enumerated(EnumType.STRING)
    private CourseStatus status = CourseStatus.DRAFT;

    private Integer durationHours;
    private Double averageRating = 0.0;
    private Integer reviewCount = 0;
    private Integer enrollmentCount = 0;

    @ElementCollection
    @CollectionTable(name = "course_requirements")
    private List<String> requirements = new ArrayList<>();

    @ElementCollection
    @CollectionTable(name = "course_outcomes")
    private List<String> learningOutcomes = new ArrayList<>();

    @OneToMany(mappedBy = "course", cascade = CascadeType.ALL)
    @OrderBy("sectionOrder ASC")
    private List<CourseSection> sections = new ArrayList<>();

    @CreatedDate
    private LocalDateTime createdAt;
}

// CourseSection.java
@Entity
@Table(name = "course_sections")
@Data
@NoArgsConstructor
public class CourseSection {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    private Course course;

    private String title;
    private Integer sectionOrder;

    @OneToMany(mappedBy = "section", cascade = CascadeType.ALL)
    @OrderBy("lessonOrder ASC")
    private List<Lesson> lessons = new ArrayList<>();
}

// Lesson.java
@Entity
@Table(name = "lessons")
@Data
@NoArgsConstructor
public class Lesson {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "section_id")
    private CourseSection section;

    private String title;

    @Enumerated(EnumType.STRING)
    private LessonType type; // VIDEO, ARTICLE, QUIZ, ASSIGNMENT

    @Column(columnDefinition = "TEXT")
    private String content;

    private String videoUrl;
    private Integer durationMinutes;

    private Boolean isPreview = false;
    private Integer lessonOrder;

    @OneToOne(mappedBy = "lesson", cascade = CascadeType.ALL)
    private Quiz quiz;
}

// Enrollment.java
@Entity
@Table(name = "course_enrollments",
       uniqueConstraints = @UniqueConstraint(columnNames = {"course_id", "student_id"}))
@Data
@NoArgsConstructor
public class CourseEnrollment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    private Course course;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private User student;

    @Column(precision = 10, scale = 2)
    private BigDecimal amountPaid;

    @Enumerated(EnumType.STRING)
    private EnrollmentStatus status = EnrollmentStatus.ACTIVE;

    private LocalDateTime completedAt;
    private Double completionPercentage = 0.0;

    @CreatedDate
    private LocalDateTime enrolledAt;
}

// LessonProgress.java
@Entity
@Table(name = "lesson_progress",
       uniqueConstraints = @UniqueConstraint(columnNames = {"enrollment_id", "lesson_id"}))
@Data
@NoArgsConstructor
public class LessonProgress {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "enrollment_id")
    private CourseEnrollment enrollment;

    @ManyToOne
    @JoinColumn(name = "lesson_id")
    private Lesson lesson;

    private Boolean completed = false;
    private Integer watchedSeconds;
    private LocalDateTime completedAt;
    private LocalDateTime lastAccessedAt;
}

// Quiz.java
@Entity
@Table(name = "quizzes")
@Data
@NoArgsConstructor
public class Quiz {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "lesson_id")
    private Lesson lesson;

    private String title;
    private Integer passingScore; // Percentage
    private Integer timeLimit; // Minutes
    private Integer maxAttempts;

    @OneToMany(mappedBy = "quiz", cascade = CascadeType.ALL)
    private List<QuizQuestion> questions = new ArrayList<>();
}

// Certificate.java
@Entity
@Table(name = "certificates")
@Data
@NoArgsConstructor
public class Certificate {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String certificateNumber;

    @ManyToOne
    @JoinColumn(name = "enrollment_id")
    private CourseEnrollment enrollment;

    private String studentName;
    private String courseName;
    private String instructorName;
    private String certificateUrl;

    @CreatedDate
    private LocalDateTime issuedAt;
}

public enum CourseStatus { DRAFT, REVIEW, PUBLISHED, ARCHIVED }
public enum LessonType { VIDEO, ARTICLE, QUIZ, ASSIGNMENT }
```

### Service Layer

```java
// LMSService.java
@Service
@Transactional
@RequiredArgsConstructor
public class LMSService {

    private final CourseRepository courseRepository;
    private final CourseEnrollmentRepository enrollmentRepository;
    private final LessonProgressRepository progressRepository;
    private final CertificateRepository certificateRepository;
    private final LessonRepository lessonRepository;

    public EnrollmentDTO enrollInCourse(Long studentId, Long courseId) {
        Course course = courseRepository.findById(courseId)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found"));

        if (course.getStatus() != CourseStatus.PUBLISHED) {
            throw new BadRequestException("Course is not available for enrollment");
        }

        if (enrollmentRepository.existsByCourseIdAndStudentId(courseId, studentId)) {
            throw new ConflictException("Already enrolled in this course");
        }

        CourseEnrollment enrollment = new CourseEnrollment();
        enrollment.setCourse(course);
        User student = new User();
        student.setId(studentId);
        enrollment.setStudent(student);
        enrollment.setAmountPaid(course.getIsFree() ? BigDecimal.ZERO : course.getPrice());

        course.setEnrollmentCount(course.getEnrollmentCount() + 1);
        courseRepository.save(course);

        return toEnrollmentDTO(enrollmentRepository.save(enrollment));
    }

    public LessonProgressDTO completeLesson(Long enrollmentId, Long lessonId) {
        CourseEnrollment enrollment = enrollmentRepository.findById(enrollmentId)
            .orElseThrow(() -> new ResourceNotFoundException("Enrollment not found"));

        Lesson lesson = lessonRepository.findById(lessonId)
            .orElseThrow(() -> new ResourceNotFoundException("Lesson not found"));

        LessonProgress progress = progressRepository
            .findByEnrollmentIdAndLessonId(enrollmentId, lessonId)
            .orElse(new LessonProgress());

        progress.setEnrollment(enrollment);
        progress.setLesson(lesson);
        progress.setCompleted(true);
        progress.setCompletedAt(LocalDateTime.now());
        progress.setLastAccessedAt(LocalDateTime.now());

        progressRepository.save(progress);

        // Update course completion percentage
        updateCourseProgress(enrollment);

        return LessonProgressDTO.builder()
            .lessonId(lessonId)
            .completed(true)
            .completedAt(progress.getCompletedAt())
            .build();
    }

    public CourseProgressDTO getCourseProgress(Long enrollmentId) {
        CourseEnrollment enrollment = enrollmentRepository.findById(enrollmentId)
            .orElseThrow(() -> new ResourceNotFoundException("Enrollment not found"));

        Course course = enrollment.getCourse();
        int totalLessons = course.getSections().stream()
            .mapToInt(s -> s.getLessons().size()).sum();

        List<Long> completedIds = progressRepository.findByEnrollmentId(enrollmentId).stream()
            .filter(LessonProgress::getCompleted)
            .map(p -> p.getLesson().getId())
            .collect(Collectors.toList());

        double percentage = totalLessons > 0 ?
            (double) completedIds.size() / totalLessons * 100 : 0;

        return CourseProgressDTO.builder()
            .enrollmentId(enrollmentId)
            .courseName(course.getTitle())
            .totalLessons(totalLessons)
            .completedLessons(completedIds.size())
            .progressPercentage(Math.round(percentage * 100.0) / 100.0)
            .completedAt(enrollment.getCompletedAt())
            .certificate(enrollment.getCompletedAt() != null ?
                getCertificateForEnrollment(enrollmentId) : null)
            .build();
    }

    private void updateCourseProgress(CourseEnrollment enrollment) {
        Course course = enrollment.getCourse();
        int totalLessons = course.getSections().stream()
            .mapToInt(s -> s.getLessons().size()).sum();

        long completedLessons = progressRepository.countByEnrollmentIdAndCompleted(enrollment.getId(), true);

        double percentage = totalLessons > 0 ? (double) completedLessons / totalLessons * 100 : 0;
        enrollment.setCompletionPercentage(percentage);

        if (percentage >= 100.0) {
            enrollment.setCompletedAt(LocalDateTime.now());
            issueCertificate(enrollment);
        }

        enrollmentRepository.save(enrollment);
    }

    private void issueCertificate(CourseEnrollment enrollment) {
        if (certificateRepository.existsByEnrollmentId(enrollment.getId())) {
            return; // Already issued
        }

        Certificate cert = new Certificate();
        cert.setCertificateNumber("CERT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase());
        cert.setEnrollment(enrollment);
        cert.setStudentName(enrollment.getStudent().getFullName());
        cert.setCourseName(enrollment.getCourse().getTitle());
        cert.setInstructorName(enrollment.getCourse().getInstructor().getFullName());

        // Generate certificate PDF/image
        // String certUrl = certificateGeneratorService.generate(cert);
        // cert.setCertificateUrl(certUrl);

        certificateRepository.save(cert);
    }

    private CertificateDTO getCertificateForEnrollment(Long enrollmentId) {
        return certificateRepository.findByEnrollmentId(enrollmentId)
            .map(c -> CertificateDTO.builder()
                .certificateNumber(c.getCertificateNumber())
                .courseName(c.getCourseName())
                .issuedAt(c.getIssuedAt())
                .certificateUrl(c.getCertificateUrl())
                .build())
            .orElse(null);
    }

    private EnrollmentDTO toEnrollmentDTO(CourseEnrollment e) {
        return EnrollmentDTO.builder()
            .id(e.getId())
            .courseTitle(e.getCourse().getTitle())
            .instructorName(e.getCourse().getInstructor().getUsername())
            .status(e.getStatus())
            .completionPercentage(e.getCompletionPercentage())
            .enrolledAt(e.getEnrolledAt())
            .build();
    }
}
```

### REST Controller (LMS)

```java
// LMSController.java
@RestController
@RequestMapping("/api/v1/lms")
@RequiredArgsConstructor
public class LMSController {

    private final LMSService lmsService;

    @GetMapping("/courses")
    public ResponseEntity<Page<CourseDTO>> getCourses(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String level,
            @RequestParam(required = false) String search,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(lmsService.getCourses(category, level, search, PageRequest.of(page, size)));
    }

    @GetMapping("/courses/{id}")
    public ResponseEntity<CourseDetailDTO> getCourse(@PathVariable Long id) {
        return ResponseEntity.ok(lmsService.getCourseDetail(id));
    }

    @PostMapping("/courses/{id}/enroll")
    public ResponseEntity<EnrollmentDTO> enroll(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(lmsService.enrollInCourse(user.getId(), id));
    }

    @GetMapping("/my-courses")
    public ResponseEntity<List<EnrollmentDTO>> getMyCourses(@AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(lmsService.getMyEnrollments(user.getId()));
    }

    @PostMapping("/enrollments/{enrollmentId}/lessons/{lessonId}/complete")
    public ResponseEntity<LessonProgressDTO> completeLesson(
            @PathVariable Long enrollmentId,
            @PathVariable Long lessonId,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(lmsService.completeLesson(enrollmentId, lessonId));
    }

    @GetMapping("/enrollments/{enrollmentId}/progress")
    public ResponseEntity<CourseProgressDTO> getProgress(@PathVariable Long enrollmentId) {
        return ResponseEntity.ok(lmsService.getCourseProgress(enrollmentId));
    }

    @GetMapping("/certificates")
    public ResponseEntity<List<CertificateDTO>> getMyCertificates(
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(lmsService.getMyCertificates(user.getId()));
    }

    // Instructor endpoints
    @PostMapping("/instructor/courses")
    @PreAuthorize("hasRole('INSTRUCTOR')")
    public ResponseEntity<CourseDTO> createCourse(
            @Valid @RequestBody CreateCourseRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.status(HttpStatus.CREATED).body(lmsService.createCourse(request, user.getId()));
    }

    @PutMapping("/instructor/courses/{id}")
    @PreAuthorize("hasRole('INSTRUCTOR')")
    public ResponseEntity<CourseDTO> updateCourse(
            @PathVariable Long id,
            @Valid @RequestBody UpdateCourseRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(lmsService.updateCourse(id, request, user.getId()));
    }

    @PostMapping("/instructor/courses/{id}/publish")
    @PreAuthorize("hasRole('INSTRUCTOR') or hasRole('ADMIN')")
    public ResponseEntity<CourseDTO> publishCourse(@PathVariable Long id) {
        return ResponseEntity.ok(lmsService.publishCourse(id));
    }
}
```

### Flyway Migration

```sql
-- V20__init_lms.sql
CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    short_description TEXT,
    instructor_id BIGINT NOT NULL,
    category VARCHAR(100),
    level VARCHAR(20),
    language VARCHAR(50) DEFAULT 'Thai',
    thumbnail_url VARCHAR(500),
    preview_video_url VARCHAR(500),
    price DECIMAL(10,2),
    is_free BOOLEAN DEFAULT FALSE,
    status VARCHAR(20) DEFAULT 'DRAFT',
    duration_hours INT,
    average_rating DOUBLE PRECISION DEFAULT 0,
    review_count INT DEFAULT 0,
    enrollment_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE course_sections (
    id BIGSERIAL PRIMARY KEY,
    course_id BIGINT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    section_order INT NOT NULL
);

CREATE TABLE lessons (
    id BIGSERIAL PRIMARY KEY,
    section_id BIGINT NOT NULL REFERENCES course_sections(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    type VARCHAR(20) NOT NULL,
    content TEXT,
    video_url VARCHAR(500),
    duration_minutes INT,
    is_preview BOOLEAN DEFAULT FALSE,
    lesson_order INT NOT NULL
);

CREATE TABLE course_enrollments (
    id BIGSERIAL PRIMARY KEY,
    course_id BIGINT NOT NULL REFERENCES courses(id),
    student_id BIGINT NOT NULL,
    amount_paid DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    completed_at TIMESTAMP,
    completion_percentage DOUBLE PRECISION DEFAULT 0,
    enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (course_id, student_id)
);

CREATE TABLE lesson_progress (
    id BIGSERIAL PRIMARY KEY,
    enrollment_id BIGINT NOT NULL REFERENCES course_enrollments(id),
    lesson_id BIGINT NOT NULL REFERENCES lessons(id),
    completed BOOLEAN DEFAULT FALSE,
    watched_seconds INT,
    completed_at TIMESTAMP,
    last_accessed_at TIMESTAMP,
    UNIQUE (enrollment_id, lesson_id)
);

CREATE TABLE certificates (
    id BIGSERIAL PRIMARY KEY,
    certificate_number VARCHAR(50) UNIQUE,
    enrollment_id BIGINT UNIQUE REFERENCES course_enrollments(id),
    student_name VARCHAR(255),
    course_name VARCHAR(255),
    instructor_name VARCHAR(255),
    certificate_url VARCHAR(500),
    issued_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ตัวอย่าง API Calls

```bash
# ดูรายการคอร์ส
curl "http://localhost:8080/api/v1/lms/courses?category=Programming&level=BEGINNER" \
  -H "Authorization: Bearer {token}"

# ลงทะเบียนคอร์ส
curl -X POST http://localhost:8080/api/v1/lms/courses/5/enroll \
  -H "Authorization: Bearer {token}"

# ดูคอร์สที่ลงทะเบียน
curl http://localhost:8080/api/v1/lms/my-courses \
  -H "Authorization: Bearer {token}"

# บันทึกว่าเรียนบทเรียนเสร็จ
curl -X POST http://localhost:8080/api/v1/lms/enrollments/1/lessons/12/complete \
  -H "Authorization: Bearer {token}"

# ดูความคืบหน้า
curl http://localhost:8080/api/v1/lms/enrollments/1/progress \
  -H "Authorization: Bearer {token}"

# ดูใบรับรอง
curl http://localhost:8080/api/v1/lms/certificates \
  -H "Authorization: Bearer {token}"

# Instructor: สร้างคอร์ส
curl -X POST http://localhost:8080/api/v1/lms/instructor/courses \
  -H "Authorization: Bearer {instructor_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Spring Boot Masterclass",
    "description": "...",
    "category": "Backend Development",
    "level": "INTERMEDIATE",
    "price": 1500.00
  }'
```

---

*[← Part 103: Operations & Management](./part-103-management-systems.md) | [Part 105: Booking & Marketplace →](./part-105-booking-marketplace.md)*

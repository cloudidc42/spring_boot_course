# Part 117: โปรเจค 81-85 — Media & Entertainment

> **ระดับ:** โลก (World-Class) | **เวลาเรียนรู้:** 10-15 ชั่วโมง  
> **เป้าหมาย:** สร้างระบบ Media & Entertainment Platforms ที่ใช้งานได้จริงในระดับ Production

---

## โปรเจค 81: Music Streaming API

### ภาพรวมโปรเจค

Music Streaming API เป็นระบบ Backend สำหรับแพลตฟอร์มฟังเพลง คล้ายกับ Spotify รองรับการจัดการ Artists, Albums, Tracks, Playlists, การสร้าง Streaming URLs แบบ Presigned S3, Play History, Recommendations และ Charts (Top Tracks/Artists)

### Entities

```java
// Artist.java — ข้อมูลศิลปิน
@Entity
@Table(name = "artists")
public class Artist {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String bio;
    private String profileImageUrl;
    private String coverImageUrl;

    @Column(nullable = false)
    private String slug; // สำหรับ URL-friendly ID

    private Long monthlyListeners;
    private Long totalFollowers;

    @OneToMany(mappedBy = "artist")
    private List<Album> albums = new ArrayList<>();

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Album.java — อัลบั้มเพลง
@Entity
@Table(name = "albums")
public class Album {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @ManyToOne
    @JoinColumn(name = "artist_id")
    private Artist artist;

    @Enumerated(EnumType.STRING)
    private AlbumType type; // ALBUM, SINGLE, EP, COMPILATION

    private String coverImageUrl;
    private LocalDate releaseDate;
    private String label; // ค่ายเพลง
    private String upc; // Universal Product Code

    @OneToMany(mappedBy = "album", cascade = CascadeType.ALL)
    @OrderBy("trackNumber ASC")
    private List<Track> tracks = new ArrayList<>();

    private Long totalPlays;
    private LocalDateTime createdAt;
}

// Track.java — เพลงแต่ละเพลง
@Entity
@Table(name = "tracks")
public class Track {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @ManyToOne
    @JoinColumn(name = "album_id")
    private Album album;

    @ManyToOne
    @JoinColumn(name = "artist_id")
    private Artist artist;

    @ManyToMany
    @JoinTable(name = "track_featuring_artists",
            joinColumns = @JoinColumn(name = "track_id"),
            inverseJoinColumns = @JoinColumn(name = "artist_id"))
    private Set<Artist> featuringArtists = new HashSet<>();

    private Integer trackNumber;
    private Integer durationSeconds;
    private String isrc; // International Standard Recording Code

    // S3 Key สำหรับไฟล์เพลง (ไม่เปิดเผยสาธารณะ)
    private String audioS3Key;
    private String previewS3Key; // 30 วินาที preview

    private String genre;
    private String bpm; // beats per minute
    private boolean explicit; // เนื้อหาไม่เหมาะสม

    private Long totalPlays;
    private Long totalLikes;

    @Enumerated(EnumType.STRING)
    private TrackStatus status; // ACTIVE, REMOVED, PENDING

    private LocalDateTime createdAt;
}

// Playlist.java — Playlist ที่ User สร้าง
@Entity
@Table(name = "playlists")
public class Playlist {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;
    private String coverImageUrl;

    @ManyToOne
    @JoinColumn(name = "owner_id")
    private User owner;

    @OneToMany(mappedBy = "playlist", cascade = CascadeType.ALL)
    @OrderBy("position ASC")
    private List<PlaylistTrack> tracks = new ArrayList<>();

    private boolean publicPlaylist;
    private Long totalFollowers;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// PlayHistory.java — ประวัติการฟังเพลง
@Entity
@Table(name = "play_history")
public class PlayHistory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "track_id")
    private Track track;

    private String context; // playlist, album, artist, search
    private Long contextId;
    private Integer playedDurationSeconds;
    private boolean completedPlay; // ฟังจบไหม

    private LocalDateTime playedAt;
}
```

### Service Layer

```java
// StreamingService.java — จัดการ Streaming URLs
@Service
public class StreamingService {

    @Autowired
    private S3Client s3Client;

    @Autowired
    private TrackRepository trackRepository;

    @Autowired
    private PlayHistoryRepository playHistoryRepository;

    @Value("${aws.s3.music-bucket}")
    private String musicBucket;

    // สร้าง Presigned URL สำหรับ Streaming
    public StreamingUrlResponse getStreamingUrl(Long trackId, Long userId) {
        Track track = trackRepository.findById(trackId)
                .orElseThrow(() -> new TrackNotFoundException("Track not found"));

        if (track.getStatus() != TrackStatus.ACTIVE) {
            throw new TrackNotAvailableException("Track is not available");
        }

        // ตรวจสอบว่า User มี subscription ไหม (ถ้าต้องการ)
        // ...

        // สร้าง Presigned URL ที่หมดอายุใน 1 ชั่วโมง
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofHours(1))
                .getObjectRequest(GetObjectRequest.builder()
                        .bucket(musicBucket)
                        .key(track.getAudioS3Key())
                        .build())
                .build();

        PresignedGetObjectRequest presigned = s3Presigner.presignGetObject(presignRequest);

        return StreamingUrlResponse.builder()
                .trackId(trackId)
                .streamUrl(presigned.url().toString())
                .expiresAt(Instant.now().plus(Duration.ofHours(1)))
                .durationSeconds(track.getDurationSeconds())
                .build();
    }

    // บันทึก Play Event
    @Async
    public void recordPlay(Long userId, Long trackId, String context, Long contextId,
                           Integer playedDuration, boolean completed) {
        PlayHistory history = new PlayHistory();
        history.setUser(new User(userId));
        history.setTrack(new Track(trackId));
        history.setContext(context);
        history.setContextId(contextId);
        history.setPlayedDurationSeconds(playedDuration);
        history.setCompletedPlay(completed);
        history.setPlayedAt(LocalDateTime.now());
        playHistoryRepository.save(history);

        // อัปเดต Play Count ของ Track
        trackRepository.incrementPlayCount(trackId);
    }
}

// ChartService.java — คำนวณ Charts
@Service
public class ChartService {

    @Autowired
    private PlayHistoryRepository playHistoryRepository;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // Top Tracks ใน 7 วันที่ผ่านมา
    public List<ChartEntry> getTopTracks(String genre, int limit) {
        String cacheKey = "charts:tracks:" + (genre != null ? genre : "all") + ":" + limit;

        // ดึงจาก Cache ก่อน
        List<ChartEntry> cached = (List<ChartEntry>) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) return cached;

        LocalDateTime since = LocalDateTime.now().minusDays(7);
        List<Object[]> results = playHistoryRepository.findTopTracks(genre, since, limit);

        List<ChartEntry> chart = new ArrayList<>();
        for (int i = 0; i < results.size(); i++) {
            Object[] row = results.get(i);
            chart.add(ChartEntry.builder()
                    .position(i + 1)
                    .trackId(((Number) row[0]).longValue())
                    .trackTitle((String) row[1])
                    .artistName((String) row[2])
                    .playCount(((Number) row[3]).longValue())
                    .build());
        }

        // Cache 1 ชั่วโมง
        redisTemplate.opsForValue().set(cacheKey, chart, Duration.ofHours(1));

        return chart;
    }
}

// MusicController.java
@RestController
@RequestMapping("/api/music")
public class MusicController {

    @Autowired
    private StreamingService streamingService;

    @Autowired
    private ChartService chartService;

    @Autowired
    private PlaylistService playlistService;

    @GetMapping("/tracks/{id}/stream")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<StreamingUrlResponse> getStreamingUrl(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        return ResponseEntity.ok(streamingService.getStreamingUrl(id, user.getId()));
    }

    @PostMapping("/tracks/{id}/play")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> recordPlay(
            @PathVariable Long id,
            @RequestBody PlayEventRequest request,
            @AuthenticationPrincipal User user) {
        streamingService.recordPlay(user.getId(), id, request.getContext(),
                request.getContextId(), request.getPlayedDuration(), request.isCompleted());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/charts/tracks")
    public ResponseEntity<List<ChartEntry>> topTracks(
            @RequestParam(required = false) String genre,
            @RequestParam(defaultValue = "50") int limit) {
        return ResponseEntity.ok(chartService.getTopTracks(genre, limit));
    }

    @GetMapping("/artists/{id}/top-tracks")
    public ResponseEntity<List<TrackResponse>> artistTopTracks(@PathVariable Long id) {
        return ResponseEntity.ok(streamingService.getArtistTopTracks(id, 10));
    }

    @PostMapping("/playlists")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<PlaylistResponse> createPlaylist(
            @RequestBody CreatePlaylistRequest request,
            @AuthenticationPrincipal User user) {
        Playlist playlist = playlistService.create(request, user);
        return ResponseEntity.status(HttpStatus.CREATED).body(PlaylistResponse.from(playlist));
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__music_streaming.sql
CREATE TABLE artists (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    bio TEXT,
    profile_image_url TEXT,
    cover_image_url TEXT,
    slug VARCHAR(255) UNIQUE NOT NULL,
    monthly_listeners BIGINT DEFAULT 0,
    total_followers BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE albums (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    artist_id BIGINT REFERENCES artists(id),
    type VARCHAR(50) DEFAULT 'ALBUM',
    cover_image_url TEXT,
    release_date DATE,
    label VARCHAR(255),
    upc VARCHAR(50),
    total_plays BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE tracks (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    album_id BIGINT REFERENCES albums(id),
    artist_id BIGINT REFERENCES artists(id),
    track_number INTEGER,
    duration_seconds INTEGER,
    isrc VARCHAR(50),
    audio_s3_key VARCHAR(500),
    preview_s3_key VARCHAR(500),
    genre VARCHAR(100),
    bpm VARCHAR(10),
    explicit BOOLEAN DEFAULT FALSE,
    total_plays BIGINT DEFAULT 0,
    total_likes BIGINT DEFAULT 0,
    status VARCHAR(50) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE playlists (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    cover_image_url TEXT,
    owner_id BIGINT NOT NULL,
    public_playlist BOOLEAN DEFAULT TRUE,
    total_followers BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE playlist_tracks (
    playlist_id BIGINT REFERENCES playlists(id) ON DELETE CASCADE,
    track_id BIGINT REFERENCES tracks(id),
    position INTEGER NOT NULL,
    added_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (playlist_id, track_id)
);

CREATE TABLE play_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    track_id BIGINT REFERENCES tracks(id),
    context VARCHAR(50),
    context_id BIGINT,
    played_duration_seconds INTEGER,
    completed_play BOOLEAN DEFAULT FALSE,
    played_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_play_history_user ON play_history(user_id, played_at DESC);
CREATE INDEX idx_play_history_track ON play_history(track_id, played_at DESC);
CREATE INDEX idx_tracks_artist ON tracks(artist_id);
```

---

## โปรเจค 82: Podcast Platform

### ภาพรวมโปรเจค

Podcast Platform เป็นระบบจัดการ Podcasts ครบวงจร รองรับการสร้าง Shows, Episodes, สร้าง RSS Feed อัตโนมัติ, การ Subscribe, Sync Play Progress ข้ามอุปกรณ์, หมวดหมู่ (Categories), Episode Notes และ Download Tracking

### Entities

```java
// PodcastShow.java — รายการ Podcast
@Entity
@Table(name = "podcast_shows")
public class PodcastShow {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    private String description;
    private String shortDescription;
    private String coverImageUrl;

    @Column(unique = true, nullable = false)
    private String slug;

    @ManyToOne
    @JoinColumn(name = "author_id")
    private User author;

    @ManyToMany
    @JoinTable(name = "show_categories")
    private Set<PodcastCategory> categories = new HashSet<>();

    private String language;
    private String explicit; // clean, explicit
    private String copyright;
    private String websiteUrl;
    private String email; // contact email สำหรับ RSS

    private Long totalSubscribers;
    private Long totalEpisodes;
    private Long totalPlays;

    @Enumerated(EnumType.STRING)
    private ShowStatus status; // ACTIVE, PAUSED, ENDED

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// PodcastEpisode.java — ตอนของ Podcast
@Entity
@Table(name = "podcast_episodes")
public class PodcastEpisode {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "show_id")
    private PodcastShow show;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "text")
    private String description;

    @Column(columnDefinition = "text")
    private String notes; // Show notes, links, chapters

    private Integer episodeNumber;
    private Integer seasonNumber;

    @Enumerated(EnumType.STRING)
    private EpisodeType type; // FULL, TRAILER, BONUS

    // Audio file
    private String audioS3Key;
    private String audioUrl; // Public CDN URL
    private Long fileSize;
    private Integer durationSeconds;
    private String mimeType; // audio/mpeg, audio/ogg

    private Long totalPlays;
    private Long totalDownloads;

    @Enumerated(EnumType.STRING)
    private EpisodeStatus status; // DRAFT, SCHEDULED, PUBLISHED

    private LocalDateTime publishedAt;
    private LocalDateTime scheduledAt;
    private LocalDateTime createdAt;
}

// EpisodePlayProgress.java — Sync ตำแหน่งการฟัง
@Entity
@Table(name = "episode_play_progress")
public class EpisodePlayProgress {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "episode_id")
    private PodcastEpisode episode;

    private Integer positionSeconds; // ตำแหน่งล่าสุดที่ฟัง (วินาที)
    private Integer totalDurationSeconds;
    private boolean completed;

    @Column(unique = true)
    private String userEpisodeKey; // userId:episodeId สำหรับ upsert

    private LocalDateTime updatedAt;
}
```

### RSS Feed Generation

```java
// RssFeedService.java — สร้าง RSS Feed สำหรับ Podcast Players
@Service
public class RssFeedService {

    @Autowired
    private PodcastShowRepository showRepository;

    @Autowired
    private PodcastEpisodeRepository episodeRepository;

    // สร้าง RSS Feed ตาม Podcast Standard (iTunes/Apple Podcasts compatible)
    public String generateRssFeed(String showSlug) {
        PodcastShow show = showRepository.findBySlug(showSlug)
                .orElseThrow(() -> new ShowNotFoundException("Show not found: " + showSlug));

        List<PodcastEpisode> episodes = episodeRepository.findByShowAndStatusOrderByPublishedAtDesc(
                show, EpisodeStatus.PUBLISHED);

        SyndFeed feed = new SyndFeedImpl();
        feed.setFeedType("rss_2.0");
        feed.setTitle(show.getTitle());
        feed.setDescription(show.getDescription());
        feed.setLink(show.getWebsiteUrl());
        feed.setLanguage(show.getLanguage() != null ? show.getLanguage() : "en");

        // iTunes-specific modules
        EntryInformation itunesModule = new EntryInformationImpl();
        itunesModule.setAuthor(show.getAuthor().getName());
        itunesModule.setExplicit(show.getExplicit() != null && show.getExplicit().equals("explicit"));
        itunesModule.setImage(show.getCoverImageUrl());
        itunesModule.setCategories(buildItunesCategories(show.getCategories()));
        feed.getModules().add(itunesModule);

        // สร้าง Episodes เป็น Feed Entries
        List<SyndEntry> entries = new ArrayList<>();
        for (PodcastEpisode episode : episodes) {
            SyndEntry entry = new SyndEntryImpl();
            entry.setTitle(episode.getTitle());
            entry.setDescription(new SyndContentImpl());
            entry.getDescription().setValue(episode.getDescription());
            entry.setPublishedDate(Date.from(episode.getPublishedAt().toInstant(ZoneOffset.UTC)));

            // Audio enclosure (สำคัญมากสำหรับ Podcast)
            SyndEnclosure enclosure = new SyndEnclosureImpl();
            enclosure.setUrl(episode.getAudioUrl());
            enclosure.setType(episode.getMimeType() != null ? episode.getMimeType() : "audio/mpeg");
            enclosure.setLength(episode.getFileSize() != null ? episode.getFileSize() : 0L);
            entry.getEnclosures().add(enclosure);

            // iTunes episode info
            EntryInformation episodeItunes = new EntryInformationImpl();
            episodeItunes.setDuration(formatDuration(episode.getDurationSeconds()));
            episodeItunes.setEpisodeType(episode.getType().name().toLowerCase());
            episodeItunes.setEpisode(episode.getEpisodeNumber());
            episodeItunes.setSeason(episode.getSeasonNumber());
            entry.getModules().add(episodeItunes);

            entries.add(entry);
        }
        feed.setEntries(entries);

        // Convert to RSS XML string
        try {
            WireFeedOutput output = new WireFeedOutput();
            return output.outputString(feed);
        } catch (Exception e) {
            throw new RssFeedGenerationException("Failed to generate RSS feed", e);
        }
    }

    private String formatDuration(Integer seconds) {
        if (seconds == null) return "00:00:00";
        int h = seconds / 3600;
        int m = (seconds % 3600) / 60;
        int s = seconds % 60;
        return String.format("%02d:%02d:%02d", h, m, s);
    }
}

// PodcastService.java — จัดการ Podcast Shows & Episodes
@Service
@Transactional
public class PodcastService {

    @Autowired
    private PodcastShowRepository showRepository;

    @Autowired
    private PodcastEpisodeRepository episodeRepository;

    @Autowired
    private EpisodePlayProgressRepository progressRepository;

    @Autowired
    private S3Client s3Client;

    // อัปโหลด Audio File สำหรับ Episode
    public PodcastEpisode uploadEpisodeAudio(Long episodeId, MultipartFile file, Long userId) throws IOException {
        PodcastEpisode episode = episodeRepository.findById(episodeId)
                .orElseThrow(() -> new EpisodeNotFoundException("Episode not found"));

        // ตรวจสอบว่าเจ้าของ Show เป็น User คนนี้
        if (!episode.getShow().getAuthor().getId().equals(userId)) {
            throw new AccessDeniedException("You don't own this show");
        }

        String s3Key = "podcasts/" + episode.getShow().getSlug() + "/episodes/" +
                episodeId + "/" + UUID.randomUUID() + getExtension(file.getOriginalFilename());

        // Upload to S3
        s3Client.putObject(PutObjectRequest.builder()
                        .bucket(podcastBucket)
                        .key(s3Key)
                        .contentType(file.getContentType())
                        .contentLength(file.getSize())
                        .build(),
                RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

        episode.setAudioS3Key(s3Key);
        episode.setAudioUrl(cdnBaseUrl + "/" + s3Key);
        episode.setFileSize(file.getSize());
        episode.setMimeType(file.getContentType());

        return episodeRepository.save(episode);
    }

    // Sync Play Progress
    public void syncPlayProgress(Long userId, Long episodeId, int positionSeconds, boolean completed) {
        String key = userId + ":" + episodeId;

        EpisodePlayProgress progress = progressRepository.findByUserEpisodeKey(key)
                .orElse(new EpisodePlayProgress());

        progress.setUser(new User(userId));
        progress.setEpisode(new PodcastEpisode(episodeId));
        progress.setPositionSeconds(positionSeconds);
        progress.setCompleted(completed);
        progress.setUserEpisodeKey(key);
        progress.setUpdatedAt(LocalDateTime.now());

        progressRepository.save(progress);

        // เพิ่ม play count เมื่อฟังจบ
        if (completed) {
            episodeRepository.incrementPlayCount(episodeId);
        }
    }

    // บันทึก Download
    @Async
    public void trackDownload(Long episodeId, String ipAddress, String userAgent) {
        episodeRepository.incrementDownloadCount(episodeId);
        // บันทึก download log สำหรับ analytics
    }
}

// PodcastController.java
@RestController
@RequestMapping("/api/podcasts")
public class PodcastController {

    @Autowired
    private PodcastService podcastService;

    @Autowired
    private RssFeedService rssFeedService;

    @GetMapping("/{slug}/feed.xml")
    public ResponseEntity<String> getRssFeed(@PathVariable String slug) {
        String feed = rssFeedService.generateRssFeed(slug);
        return ResponseEntity.ok()
                .contentType(MediaType.APPLICATION_RSS_XML)
                .body(feed);
    }

    @PostMapping("/shows")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ShowResponse> createShow(
            @RequestBody CreateShowRequest request,
            @AuthenticationPrincipal User user) {
        PodcastShow show = podcastService.createShow(request, user);
        return ResponseEntity.status(HttpStatus.CREATED).body(ShowResponse.from(show));
    }

    @PostMapping("/episodes/{id}/audio")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<EpisodeResponse> uploadAudio(
            @PathVariable Long id,
            @RequestParam("file") MultipartFile file,
            @AuthenticationPrincipal User user) throws IOException {
        PodcastEpisode episode = podcastService.uploadEpisodeAudio(id, file, user.getId());
        return ResponseEntity.ok(EpisodeResponse.from(episode));
    }

    @PostMapping("/episodes/{id}/progress")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> syncProgress(
            @PathVariable Long id,
            @RequestBody PlayProgressRequest request,
            @AuthenticationPrincipal User user) {
        podcastService.syncPlayProgress(user.getId(), id, request.getPosition(), request.isCompleted());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/episodes/{id}/progress")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<PlayProgressResponse> getProgress(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        return ResponseEntity.ok(podcastService.getPlayProgress(user.getId(), id));
    }
}
```

---

## โปรเจค 83: Digital Library

### ภาพรวมโปรเจค

Digital Library เป็นระบบห้องสมุดดิจิทัลที่รองรับการยืมหนังสืออิเล็กทรอนิกส์ (PDF/EPUB), Sync Reading Progress ข้ามอุปกรณ์, Highlights & Notes, Collections, แนวคิด DRM สำหรับป้องกันการ copy และ Recommendation System

### Entities

```java
// Book.java — หนังสือในห้องสมุด
@Entity
@Table(name = "books")
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @ManyToMany
    @JoinTable(name = "book_authors")
    private Set<Author> authors = new HashSet<>();

    private String isbn;
    private String description;
    private String coverImageUrl;
    private String publisher;
    private Integer publishYear;
    private String language;
    private Integer totalPages;

    @ManyToMany
    @JoinTable(name = "book_categories")
    private Set<BookCategory> categories = new HashSet<>();

    // File storage
    private String pdfS3Key;
    private String epubS3Key;

    @Enumerated(EnumType.STRING)
    private BookFormat availableFormats; // PDF, EPUB, BOTH

    // Borrowing settings
    private Integer maxConcurrentBorrows; // กี่ User ยืมได้พร้อมกัน
    private Integer borrowDurationDays;   // ยืมได้กี่วัน
    private Integer currentBorrowCount;

    private Float averageRating;
    private Long totalRatings;
    private Long totalReads;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// BookBorrow.java — การยืมหนังสือ
@Entity
@Table(name = "book_borrows")
public class BookBorrow {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "book_id")
    private Book book;

    @Enumerated(EnumType.STRING)
    private BookFormat format; // PDF หรือ EPUB

    @Enumerated(EnumType.STRING)
    private BorrowStatus status; // ACTIVE, RETURNED, EXPIRED

    private LocalDateTime borrowedAt;
    private LocalDateTime dueDate;
    private LocalDateTime returnedAt;

    // DRM Token สำหรับ access control
    private String drmToken;
    private LocalDateTime drmTokenExpiry;
}

// ReadingProgress.java — ติดตามความคืบหน้าการอ่าน
@Entity
@Table(name = "reading_progress")
public class ReadingProgress {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "book_id")
    private Book book;

    private Integer currentPage;
    private Float percentageRead;
    private String cfiLocation; // EPUB CFI location string

    private String deviceId;
    private String deviceName;

    private LocalDateTime lastReadAt;
    private LocalDateTime updatedAt;
}

// Highlight.java — ข้อความที่ Highlight ในหนังสือ
@Entity
@Table(name = "highlights")
public class Highlight {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "book_id")
    private Book book;

    @Column(columnDefinition = "text", nullable = false)
    private String text; // ข้อความที่ highlight

    private String cfiStart; // EPUB CFI start position
    private String cfiEnd;   // EPUB CFI end position
    private Integer pageNumber;

    @Column(name = "highlight_color")
    private String color; // yellow, blue, green, pink

    @Column(columnDefinition = "text")
    private String note; // Note ที่แนบกับ highlight

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### Service Layer

```java
// DigitalLibraryService.java — จัดการการยืม-คืนหนังสือ
@Service
@Transactional
public class DigitalLibraryService {

    @Autowired
    private BookRepository bookRepository;

    @Autowired
    private BookBorrowRepository borrowRepository;

    @Autowired
    private ReadingProgressRepository progressRepository;

    @Autowired
    private S3Presigner s3Presigner;

    // ยืมหนังสือ
    public BookBorrow borrowBook(Long userId, Long bookId, BookFormat format) {
        Book book = bookRepository.findById(bookId)
                .orElseThrow(() -> new BookNotFoundException("Book not found"));

        // ตรวจสอบว่ายังมี slot ว่างไหม
        if (book.getCurrentBorrowCount() >= book.getMaxConcurrentBorrows()) {
            throw new BookNotAvailableException("Book is currently fully borrowed");
        }

        // ตรวจสอบว่า User ยืมอยู่แล้วหรือเปล่า
        if (borrowRepository.existsByUserIdAndBookIdAndStatus(userId, bookId, BorrowStatus.ACTIVE)) {
            throw new AlreadyBorrowedException("You already have this book");
        }

        // สร้าง DRM Token
        String drmToken = generateDrmToken(userId, bookId);

        BookBorrow borrow = new BookBorrow();
        borrow.setUser(new User(userId));
        borrow.setBook(book);
        borrow.setFormat(format);
        borrow.setStatus(BorrowStatus.ACTIVE);
        borrow.setBorrowedAt(LocalDateTime.now());
        borrow.setDueDate(LocalDateTime.now().plusDays(book.getBorrowDurationDays()));
        borrow.setDrmToken(drmToken);
        borrow.setDrmTokenExpiry(LocalDateTime.now().plusDays(book.getBorrowDurationDays()));

        borrow = borrowRepository.save(borrow);

        // อัปเดต borrow count
        bookRepository.incrementBorrowCount(bookId);

        return borrow;
    }

    // ดาวน์โหลดหนังสือ (ตรวจสอบ DRM Token)
    public DownloadUrlResponse getDownloadUrl(Long userId, Long borrowId) {
        BookBorrow borrow = borrowRepository.findById(borrowId)
                .orElseThrow(() -> new BorrowNotFoundException("Borrow not found"));

        if (!borrow.getUser().getId().equals(userId)) {
            throw new AccessDeniedException("This borrow doesn't belong to you");
        }

        if (borrow.getStatus() != BorrowStatus.ACTIVE) {
            throw new BorrowExpiredException("This borrow is no longer active");
        }

        if (borrow.getDueDate().isBefore(LocalDateTime.now())) {
            // Auto-expire
            expireBorrow(borrow);
            throw new BorrowExpiredException("Borrow period has expired");
        }

        // สร้าง Presigned URL
        String s3Key = borrow.getFormat() == BookFormat.PDF ?
                borrow.getBook().getPdfS3Key() :
                borrow.getBook().getEpubS3Key();

        String presignedUrl = createPresignedUrl(s3Key, 30); // 30 นาที

        return DownloadUrlResponse.builder()
                .url(presignedUrl)
                .format(borrow.getFormat().name())
                .expiresInMinutes(30)
                .drmToken(borrow.getDrmToken())
                .build();
    }

    // Sync Reading Progress
    public void syncProgress(Long userId, Long bookId, int currentPage,
                              float percentage, String cfiLocation, String deviceId) {
        ReadingProgress progress = progressRepository
                .findByUserIdAndBookId(userId, bookId)
                .orElse(new ReadingProgress());

        progress.setUser(new User(userId));
        progress.setBook(new Book(bookId));
        progress.setCurrentPage(currentPage);
        progress.setPercentageRead(percentage);
        progress.setCfiLocation(cfiLocation);
        progress.setDeviceId(deviceId);
        progress.setLastReadAt(LocalDateTime.now());
        progress.setUpdatedAt(LocalDateTime.now());

        progressRepository.save(progress);
    }

    private String generateDrmToken(Long userId, Long bookId) {
        return UUID.randomUUID().toString() + "-" + userId + "-" + bookId;
    }
}

// BookController.java
@RestController
@RequestMapping("/api/library")
public class BookController {

    @Autowired
    private DigitalLibraryService libraryService;

    @PostMapping("/books/{id}/borrow")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<BorrowResponse> borrow(
            @PathVariable Long id,
            @RequestBody BorrowRequest request,
            @AuthenticationPrincipal User user) {
        BookBorrow borrow = libraryService.borrowBook(user.getId(), id, request.getFormat());
        return ResponseEntity.status(HttpStatus.CREATED).body(BorrowResponse.from(borrow));
    }

    @GetMapping("/borrows/{id}/download")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<DownloadUrlResponse> getDownloadUrl(
            @PathVariable Long id,
            @AuthenticationPrincipal User user) {
        return ResponseEntity.ok(libraryService.getDownloadUrl(user.getId(), id));
    }

    @PostMapping("/books/{id}/progress")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> syncProgress(
            @PathVariable Long id,
            @RequestBody ReadingProgressRequest request,
            @AuthenticationPrincipal User user) {
        libraryService.syncProgress(user.getId(), id, request.getCurrentPage(),
                request.getPercentage(), request.getCfiLocation(), request.getDeviceId());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/highlights")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<HighlightResponse> createHighlight(
            @RequestBody CreateHighlightRequest request,
            @AuthenticationPrincipal User user) {
        Highlight highlight = libraryService.createHighlight(request, user.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(HighlightResponse.from(highlight));
    }
}
```

---

## โปรเจค 84: Short Video Platform API

### ภาพรวมโปรเจค

Short Video Platform API เป็น Backend สำหรับแพลตฟอร์มวิดีโอสั้น คล้าย TikTok รองรับการอัปโหลดวิดีโอ, Processing Pipeline แบบ Async (transcoding หลาย resolution), Feed Algorithm, Likes/Comments/Shares, Follow/Following และ Trending Content

### Entities

```java
// Video.java — วิดีโอที่ Upload
@Entity
@Table(name = "videos")
public class Video {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "creator_id")
    private User creator;

    private String title;
    private String description;

    // Original upload
    private String originalS3Key;

    // Processed versions
    private String s3Key360p;
    private String s3Key720p;
    private String s3Key1080p;
    private String thumbnailS3Key;

    private Integer durationSeconds;
    private String originalFileName;
    private Long fileSize;

    @Enumerated(EnumType.STRING)
    private VideoStatus status; // UPLOADING, PROCESSING, PUBLISHED, FAILED, REMOVED

    // Processing job
    private String transcodeJobId; // AWS MediaConvert job ID

    // Engagement
    private Long viewCount;
    private Long likeCount;
    private Long commentCount;
    private Long shareCount;

    @ElementCollection
    @CollectionTable(name = "video_hashtags")
    private Set<String> hashtags = new HashSet<>();

    private String music; // เพลงประกอบ
    private boolean allowComments;
    private boolean allowDuet;

    private LocalDateTime publishedAt;
    private LocalDateTime createdAt;
}

// VideoComment.java — ความคิดเห็นในวิดีโอ
@Entity
@Table(name = "video_comments")
public class VideoComment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "video_id")
    private Video video;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @Column(columnDefinition = "text", nullable = false)
    private String content;

    @ManyToOne
    @JoinColumn(name = "parent_id")
    private VideoComment parentComment; // สำหรับ reply

    private Long likeCount;
    private boolean pinned; // Creator pin comment

    private LocalDateTime createdAt;
}

// UserFollow.java — Follow/Following
@Entity
@Table(name = "user_follows")
public class UserFollow {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "follower_id")
    private User follower;

    @ManyToOne
    @JoinColumn(name = "following_id")
    private User following;

    private LocalDateTime followedAt;
}
```

### Video Processing Pipeline

```java
// VideoUploadService.java — จัดการการ Upload และ Processing
@Service
public class VideoUploadService {

    @Autowired
    private S3Client s3Client;

    @Autowired
    private VideoRepository videoRepository;

    @Autowired
    private VideoTranscodingService transcodingService;

    @Value("${aws.s3.video-bucket}")
    private String videoBucket;

    // Step 1: Upload original video
    public VideoUploadInitResponse initiateUpload(Long userId, String filename, Long fileSize, String contentType) {
        String s3Key = "videos/originals/" + userId + "/" + UUID.randomUUID() + getExtension(filename);

        // Multipart Upload สำหรับไฟล์ใหญ่
        CreateMultipartUploadRequest request = CreateMultipartUploadRequest.builder()
                .bucket(videoBucket)
                .key(s3Key)
                .contentType(contentType)
                .build();

        CreateMultipartUploadResponse response = s3Client.createMultipartUpload(request);

        // สร้าง Video record
        Video video = new Video();
        video.setCreator(new User(userId));
        video.setOriginalS3Key(s3Key);
        video.setOriginalFileName(filename);
        video.setFileSize(fileSize);
        video.setStatus(VideoStatus.UPLOADING);
        video.setCreatedAt(LocalDateTime.now());
        video = videoRepository.save(video);

        return VideoUploadInitResponse.builder()
                .videoId(video.getId())
                .uploadId(response.uploadId())
                .s3Key(s3Key)
                .build();
    }

    // Step 2: เมื่อ Upload เสร็จ เริ่ม Transcoding
    @Async
    public void processUploadedVideo(Long videoId) {
        Video video = videoRepository.findById(videoId).orElseThrow();
        video.setStatus(VideoStatus.PROCESSING);
        videoRepository.save(video);

        try {
            // Trigger AWS MediaConvert job
            String jobId = transcodingService.startTranscoding(video);
            video.setTranscodeJobId(jobId);
            videoRepository.save(video);

        } catch (Exception e) {
            video.setStatus(VideoStatus.FAILED);
            videoRepository.save(video);
            log.error("Failed to start transcoding for video {}", videoId, e);
        }
    }

    // Step 3: Webhook callback จาก MediaConvert
    public void handleTranscodingComplete(String jobId, Map<String, String> outputKeys) {
        Video video = videoRepository.findByTranscodeJobId(jobId)
                .orElseThrow(() -> new VideoNotFoundException("Video not found for job: " + jobId));

        video.setS3Key360p(outputKeys.get("360p"));
        video.setS3Key720p(outputKeys.get("720p"));
        video.setS3Key1080p(outputKeys.get("1080p"));
        video.setThumbnailS3Key(outputKeys.get("thumbnail"));
        video.setStatus(VideoStatus.PUBLISHED);
        video.setPublishedAt(LocalDateTime.now());

        videoRepository.save(video);
    }
}

// FeedService.java — Algorithm สำหรับ For You Page
@Service
public class FeedService {

    @Autowired
    private VideoRepository videoRepository;

    @Autowired
    private UserFollowRepository followRepository;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // สร้าง Personalized Feed
    public List<Video> getForYouFeed(Long userId, int page, int size) {
        // ดึง Videos จาก Cache ก่อน
        String cacheKey = "feed:foryou:" + userId;

        List<Long> cachedVideoIds = (List<Long>) redisTemplate.opsForValue().get(cacheKey);

        if (cachedVideoIds == null) {
            cachedVideoIds = buildForYouFeed(userId);
            redisTemplate.opsForValue().set(cacheKey, cachedVideoIds, Duration.ofMinutes(30));
        }

        // Paginate
        int start = page * size;
        int end = Math.min(start + size, cachedVideoIds.size());
        if (start >= cachedVideoIds.size()) return Collections.emptyList();

        List<Long> pageIds = cachedVideoIds.subList(start, end);
        return videoRepository.findAllById(pageIds);
    }

    private List<Long> buildForYouFeed(Long userId) {
        List<Long> feedVideoIds = new ArrayList<>();

        // 40% videos จาก accounts ที่ follow
        List<Long> followingIds = followRepository.findFollowingIds(userId);
        if (!followingIds.isEmpty()) {
            List<Video> followingVideos = videoRepository.findRecentByCreators(
                    followingIds, LocalDateTime.now().minusDays(7), PageRequest.of(0, 40));
            feedVideoIds.addAll(followingVideos.stream().map(Video::getId).toList());
        }

        // 60% trending videos (based on engagement score)
        List<Video> trendingVideos = videoRepository.findTrending(
                LocalDateTime.now().minusHours(48), PageRequest.of(0, 60));
        feedVideoIds.addAll(trendingVideos.stream().map(Video::getId).toList());

        // Shuffle เพื่อให้ feed ดู natural
        Collections.shuffle(feedVideoIds);

        return feedVideoIds;
    }

    // Trending Videos
    public List<Video> getTrending(int limit) {
        // คำนวณ Engagement Score = views*0.5 + likes*2 + comments*3 + shares*5
        return videoRepository.findByEngagementScore(
                LocalDateTime.now().minusHours(48), PageRequest.of(0, limit));
    }
}

// VideoController.java
@RestController
@RequestMapping("/api/videos")
public class VideoController {

    @Autowired
    private VideoUploadService uploadService;

    @Autowired
    private FeedService feedService;

    @PostMapping("/upload/initiate")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<VideoUploadInitResponse> initiateUpload(
            @RequestBody UploadInitRequest request,
            @AuthenticationPrincipal User user) {
        return ResponseEntity.ok(uploadService.initiateUpload(
                user.getId(), request.getFilename(), request.getFileSize(), request.getContentType()));
    }

    @PostMapping("/{id}/upload/complete")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> completeUpload(@PathVariable Long id) {
        uploadService.processUploadedVideo(id);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/feed/for-you")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<List<VideoResponse>> forYouFeed(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @AuthenticationPrincipal User user) {
        return ResponseEntity.ok(feedService.getForYouFeed(user.getId(), page, size)
                .stream().map(VideoResponse::from).toList());
    }

    @GetMapping("/trending")
    public ResponseEntity<List<VideoResponse>> trending(
            @RequestParam(defaultValue = "20") int limit) {
        return ResponseEntity.ok(feedService.getTrending(limit)
                .stream().map(VideoResponse::from).toList());
    }

    @PostMapping("/{id}/like")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> like(@PathVariable Long id, @AuthenticationPrincipal User user) {
        feedService.toggleLike(user.getId(), id);
        return ResponseEntity.ok().build();
    }
}
```

---

## โปรเจค 85: Live Streaming Backend

### ภาพรวมโปรเจค

Live Streaming Backend เป็นระบบสนับสนุนการ Live Stream รองรับการจัดการ Stream Keys, Stream Metadata, นับ Viewer Count แบบ Real-time, Chat ระหว่าง Stream, VOD Recording, Monetization (Bits/Donations) และ Stream Schedule

### Entities

```java
// StreamChannel.java — ช่อง Streaming ของ Creator
@Entity
@Table(name = "stream_channels")
public class StreamChannel {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;

    @Column(unique = true, nullable = false)
    private String channelName;

    // Stream Key สำหรับ OBS/Streaming Software
    @Column(unique = true, nullable = false)
    private String streamKey;

    private String title;
    private String description;
    private String category;
    private String thumbnailUrl;
    private boolean mature; // 18+

    @Enumerated(EnumType.STRING)
    private ChannelStatus status; // OFFLINE, LIVE, SCHEDULED

    // Stats
    private Long totalFollowers;
    private Long totalViews;
    private Long peakViewers;

    private LocalDateTime lastLiveAt;
    private LocalDateTime createdAt;
}

// LiveStream.java — Session การ Live Stream แต่ละครั้ง
@Entity
@Table(name = "live_streams")
public class LiveStream {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "channel_id")
    private StreamChannel channel;

    private String title;
    private String category;
    private String thumbnailUrl;
    private boolean mature;

    @Enumerated(EnumType.STRING)
    private StreamStatus status; // LIVE, ENDED

    private Integer currentViewers;
    private Integer peakViewers;
    private Long totalViews;
    private Long totalChatMessages;

    // VOD recording
    private String vodS3Key;
    private String vodUrl;
    private Integer vodDurationSeconds;

    private LocalDateTime startedAt;
    private LocalDateTime endedAt;
}

// StreamDonation.java — Donation ระหว่าง Stream
@Entity
@Table(name = "stream_donations")
public class StreamDonation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "stream_id")
    private LiveStream stream;

    @ManyToOne
    @JoinColumn(name = "donor_id")
    private User donor;

    private BigDecimal amount;
    private String currency;
    private String message;
    private Integer bits; // platform currency

    @Enumerated(EnumType.STRING)
    private DonationStatus status; // PENDING, COMPLETED, REFUNDED

    private String stripePaymentIntentId;
    private LocalDateTime donatedAt;
}

// StreamSchedule.java — กำหนดการ Stream ล่วงหน้า
@Entity
@Table(name = "stream_schedules")
public class StreamSchedule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "channel_id")
    private StreamChannel channel;

    private String title;
    private String description;
    private String category;
    private LocalDateTime scheduledStartAt;
    private Integer estimatedDurationMinutes;
    private boolean notifyFollowers;
    private LocalDateTime createdAt;
}
```

### Live Stream Services

```java
// StreamKeyService.java — จัดการ Stream Keys
@Service
@Transactional
public class StreamKeyService {

    @Autowired
    private StreamChannelRepository channelRepository;

    // สร้าง/Reset Stream Key
    public String regenerateStreamKey(Long userId) {
        StreamChannel channel = channelRepository.findByUserId(userId)
                .orElseThrow(() -> new ChannelNotFoundException("Channel not found"));

        String newKey = generateSecureStreamKey();
        channel.setStreamKey(newKey);
        channelRepository.save(channel);

        return newKey;
    }

    // Validate Stream Key จาก RTMP server callback
    public StreamChannel validateStreamKey(String streamKey) {
        return channelRepository.findByStreamKey(streamKey)
                .orElseThrow(() -> new InvalidStreamKeyException("Invalid stream key"));
    }

    // RTMP server callback: Stream เริ่มต้น
    public LiveStream onStreamStart(String streamKey) {
        StreamChannel channel = validateStreamKey(streamKey);

        // สร้าง LiveStream record
        LiveStream stream = new LiveStream();
        stream.setChannel(channel);
        stream.setTitle(channel.getTitle());
        stream.setCategory(channel.getCategory());
        stream.setMature(channel.isMature());
        stream.setStatus(StreamStatus.LIVE);
        stream.setCurrentViewers(0);
        stream.setPeakViewers(0);
        stream.setTotalViews(0L);
        stream.setStartedAt(LocalDateTime.now());

        stream = streamRepository.save(stream);

        // อัปเดต Channel status
        channel.setStatus(ChannelStatus.LIVE);
        channel.setLastLiveAt(LocalDateTime.now());
        channelRepository.save(channel);

        // แจ้งเตือน Followers ผ่าน WebSocket
        notifyFollowers(channel, stream);

        return stream;
    }

    // RTMP server callback: Stream จบ
    public void onStreamEnd(String streamKey) {
        StreamChannel channel = validateStreamKey(streamKey);

        LiveStream stream = streamRepository.findActiveByChannelId(channel.getId())
                .orElseThrow();

        stream.setStatus(StreamStatus.ENDED);
        stream.setEndedAt(LocalDateTime.now());
        streamRepository.save(stream);

        channel.setStatus(ChannelStatus.OFFLINE);
        if (stream.getPeakViewers() > channel.getPeakViewers()) {
            channel.setPeakViewers((long) stream.getPeakViewers());
        }
        channelRepository.save(channel);
    }

    private String generateSecureStreamKey() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return "live_" + Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}

// ViewerCountService.java — นับ Viewer แบบ Real-time ด้วย Redis
@Service
public class ViewerCountService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private LiveStreamRepository streamRepository;

    private static final String VIEWER_KEY_PREFIX = "stream:viewers:";

    // User join stream
    public void addViewer(Long streamId, Long userId) {
        String key = VIEWER_KEY_PREFIX + streamId;
        redisTemplate.opsForSet().add(key, userId.toString());
        redisTemplate.expire(key, Duration.ofHours(24));

        // อัปเดต peak viewers ถ้าจำเป็น
        long currentCount = getCurrentViewerCount(streamId);
        streamRepository.updateViewersIfPeak(streamId, (int) currentCount);
    }

    // User leave stream
    public void removeViewer(Long streamId, Long userId) {
        String key = VIEWER_KEY_PREFIX + streamId;
        redisTemplate.opsForSet().remove(key, userId.toString());
    }

    // ดึงจำนวน Viewer ปัจจุบัน
    public long getCurrentViewerCount(Long streamId) {
        String key = VIEWER_KEY_PREFIX + streamId;
        Long size = redisTemplate.opsForSet().size(key);
        return size != null ? size : 0L;
    }
}

// StreamChatService.java — Chat ระหว่าง Stream ผ่าน WebSocket
@Service
public class StreamChatService {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    @Autowired
    private StreamChatMessageRepository chatRepository;

    // ส่ง Chat Message
    public ChatMessage sendMessage(Long streamId, Long userId, String username,
                                    String content, String badgeType) {
        // Validate content (filter inappropriate content)
        content = filterContent(content);

        ChatMessage message = new ChatMessage();
        message.setStreamId(streamId);
        message.setUserId(userId);
        message.setUsername(username);
        message.setContent(content);
        message.setBadgeType(badgeType); // subscriber, moderator, etc.
        message.setSentAt(LocalDateTime.now());
        chatRepository.save(message);

        // Broadcast ผ่าน WebSocket
        messagingTemplate.convertAndSend(
                "/topic/stream/" + streamId + "/chat",
                ChatMessageResponse.from(message)
        );

        return message;
    }

    // ส่ง Donation Alert
    public void broadcastDonation(Long streamId, StreamDonation donation) {
        DonationAlertMessage alert = DonationAlertMessage.builder()
                .donorName(donation.getDonor().getUsername())
                .amount(donation.getAmount())
                .currency(donation.getCurrency())
                .message(donation.getMessage())
                .bits(donation.getBits())
                .build();

        messagingTemplate.convertAndSend(
                "/topic/stream/" + streamId + "/donations",
                alert
        );
    }
}

// LiveStreamController.java
@RestController
@RequestMapping("/api/streams")
public class LiveStreamController {

    @Autowired
    private StreamKeyService streamKeyService;

    @Autowired
    private ViewerCountService viewerCountService;

    @Autowired
    private StreamChatService chatService;

    // RTMP Server callback: Stream start
    @PostMapping("/rtmp/on-publish")
    public ResponseEntity<Void> onPublish(@RequestBody RtmpCallbackRequest request) {
        streamKeyService.onStreamStart(request.getStreamKey());
        return ResponseEntity.ok().build();
    }

    // RTMP Server callback: Stream end
    @PostMapping("/rtmp/on-done")
    public ResponseEntity<Void> onDone(@RequestBody RtmpCallbackRequest request) {
        streamKeyService.onStreamEnd(request.getStreamKey());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/{id}/viewer-count")
    public ResponseEntity<ViewerCountResponse> getViewerCount(@PathVariable Long id) {
        long count = viewerCountService.getCurrentViewerCount(id);
        return ResponseEntity.ok(new ViewerCountResponse(id, count));
    }

    @PostMapping("/{id}/join")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> joinStream(@PathVariable Long id, @AuthenticationPrincipal User user) {
        viewerCountService.addViewer(id, user.getId());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{id}/leave")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> leaveStream(@PathVariable Long id, @AuthenticationPrincipal User user) {
        viewerCountService.removeViewer(id, user.getId());
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{id}/chat")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<ChatMessageResponse> sendChat(
            @PathVariable Long id,
            @RequestBody SendChatRequest request,
            @AuthenticationPrincipal User user) {
        ChatMessage msg = chatService.sendMessage(id, user.getId(),
                user.getUsername(), request.getContent(), user.getBadgeType());
        return ResponseEntity.ok(ChatMessageResponse.from(msg));
    }

    @PostMapping("/{id}/donate")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<DonationResponse> donate(
            @PathVariable Long id,
            @RequestBody DonationRequest request,
            @AuthenticationPrincipal User user) {
        StreamDonation donation = donationService.processDonation(id, user.getId(), request);
        return ResponseEntity.ok(DonationResponse.from(donation));
    }
}
```

### SQL Scripts

```sql
-- migrations/V1__live_streaming.sql
CREATE TABLE stream_channels (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL,
    channel_name VARCHAR(255) UNIQUE NOT NULL,
    stream_key VARCHAR(255) UNIQUE NOT NULL,
    title VARCHAR(500),
    description TEXT,
    category VARCHAR(100),
    thumbnail_url TEXT,
    mature BOOLEAN DEFAULT FALSE,
    status VARCHAR(50) DEFAULT 'OFFLINE',
    total_followers BIGINT DEFAULT 0,
    total_views BIGINT DEFAULT 0,
    peak_viewers BIGINT DEFAULT 0,
    last_live_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE live_streams (
    id BIGSERIAL PRIMARY KEY,
    channel_id BIGINT REFERENCES stream_channels(id),
    title VARCHAR(500),
    category VARCHAR(100),
    thumbnail_url TEXT,
    mature BOOLEAN DEFAULT FALSE,
    status VARCHAR(50) DEFAULT 'LIVE',
    current_viewers INTEGER DEFAULT 0,
    peak_viewers INTEGER DEFAULT 0,
    total_views BIGINT DEFAULT 0,
    total_chat_messages BIGINT DEFAULT 0,
    vod_s3_key VARCHAR(500),
    vod_url TEXT,
    vod_duration_seconds INTEGER,
    started_at TIMESTAMP DEFAULT NOW(),
    ended_at TIMESTAMP
);

CREATE TABLE stream_donations (
    id BIGSERIAL PRIMARY KEY,
    stream_id BIGINT REFERENCES live_streams(id),
    donor_id BIGINT NOT NULL,
    amount DECIMAL(10,2),
    currency VARCHAR(3) DEFAULT 'USD',
    message TEXT,
    bits INTEGER DEFAULT 0,
    status VARCHAR(50) DEFAULT 'PENDING',
    stripe_payment_intent_id VARCHAR(255),
    donated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stream_schedules (
    id BIGSERIAL PRIMARY KEY,
    channel_id BIGINT REFERENCES stream_channels(id),
    title VARCHAR(500),
    description TEXT,
    category VARCHAR(100),
    scheduled_start_at TIMESTAMP NOT NULL,
    estimated_duration_minutes INTEGER,
    notify_followers BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stream_chat_messages (
    id BIGSERIAL PRIMARY KEY,
    stream_id BIGINT REFERENCES live_streams(id),
    user_id BIGINT NOT NULL,
    username VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    badge_type VARCHAR(50),
    sent_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_live_streams_channel ON live_streams(channel_id, status);
CREATE INDEX idx_stream_chat_stream ON stream_chat_messages(stream_id, sent_at DESC);
```

---

## สรุป Part 117

ใน Part นี้เราได้สร้าง Media & Entertainment Platforms ระดับ Production:

| โปรเจค | เทคโนโลยีหลัก | ความยาก |
|--------|---------------|---------|
| 81. Music Streaming | S3 Presigned URLs, Play Analytics | ⭐⭐⭐⭐ |
| 82. Podcast Platform | RSS Feed, S3 Upload, Progress Sync | ⭐⭐⭐⭐ |
| 83. Digital Library | DRM Concept, EPUB CFI, Borrowing | ⭐⭐⭐⭐ |
| 84. Short Video Platform | Async Transcoding, Feed Algorithm | ⭐⭐⭐⭐⭐ |
| 85. Live Streaming | RTMP Callbacks, Redis Viewer Count, WebSocket Chat | ⭐⭐⭐⭐⭐ |

### Key Takeaways

1. **Streaming Services** — Presigned S3 URLs เป็น Pattern ที่ดีที่สุดสำหรับ protect media files
2. **RSS Feeds** — Podcast RSS ต้องรองรับ iTunes namespace extensions เพื่อให้ work กับ Podcast apps
3. **Video Processing** — ใช้ Async processing เสมอเพราะ transcoding ใช้เวลานาน
4. **Live Streaming** — Redis เป็นตัวเลือกที่ดีสำหรับ real-time viewer count เนื่องจาก fast read/write

*[← Part 116](./part-116-saas-infrastructure.md) | [Part 118 →](./part-118-workflow-automation.md)*

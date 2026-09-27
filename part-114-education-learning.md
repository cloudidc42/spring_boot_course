# Part 114: โปรเจค 66-70 — Education & Learning

**ระดับ:** ระดับโลก (World-Class)
**เวลา:** 10-15 ชั่วโมง
**เป้าหมาย:** สร้างระบบการศึกษาและการเรียนรู้ที่ครบครัน ตั้งแต่ Library Management, Language Learning App, Quiz Platform, Online Code Challenge จนถึง Book Club Platform

---

*[← Part 113: Fitness & Wellness](./part-113-fitness-wellness.md) | [Part 115: Analytics & Monitoring →](./part-115-analytics-monitoring.md)*

---

## โปรเจค 66: Library Management System

### ภาพรวมระบบ

Library Management System เป็นระบบจัดการห้องสมุดดิจิทัลที่รองรับการค้นหาหนังสือ การยืม-คืน การจอง การเรียกเก็บค่าปรับ และการส่งการแจ้งเตือนอัตโนมัติ เหมาะสำหรับห้องสมุดสาธารณะและสถาบันการศึกษา

- **Book Catalog**: จัดการข้อมูลหนังสือและค้นหา
- **Member Management**: จัดการสมาชิกห้องสมุด
- **Borrowing System**: ยืม-คืนหนังสือพร้อมบันทึก
- **Fine Management**: คำนวณและเรียกเก็บค่าปรับ
- **Reservation**: จองหนังสือที่ไม่ว่างอยู่
- **Overdue Notifications**: แจ้งเตือนหนังสือเกินกำหนด

### Flyway Migration

```sql
-- V1__create_library_tables.sql
CREATE TABLE authors (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    biography TEXT,
    birth_date DATE,
    nationality VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE books (
    id BIGSERIAL PRIMARY KEY,
    isbn VARCHAR(20) NOT NULL UNIQUE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    publisher VARCHAR(200),
    published_year INT,
    edition VARCHAR(50),
    language VARCHAR(50) DEFAULT 'Thai',
    pages INT,
    category VARCHAR(100),
    subject VARCHAR(200),
    cover_image_url VARCHAR(500),
    total_copies INT NOT NULL DEFAULT 1,
    available_copies INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE book_authors (
    book_id BIGINT NOT NULL REFERENCES books(id),
    author_id BIGINT NOT NULL REFERENCES authors(id),
    PRIMARY KEY (book_id, author_id)
);

CREATE TABLE book_copies (
    id BIGSERIAL PRIMARY KEY,
    book_id BIGINT NOT NULL REFERENCES books(id),
    copy_number VARCHAR(50) NOT NULL,
    condition VARCHAR(20) DEFAULT 'GOOD',
    location VARCHAR(100),
    status VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE library_members (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    member_number VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    address TEXT,
    membership_type VARCHAR(20) NOT NULL DEFAULT 'STANDARD',
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    membership_expiry DATE,
    max_borrow_limit INT NOT NULL DEFAULT 5,
    current_borrows INT NOT NULL DEFAULT 0,
    total_fines DECIMAL(10,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE borrowing_records (
    id BIGSERIAL PRIMARY KEY,
    member_id BIGINT NOT NULL REFERENCES library_members(id),
    book_copy_id BIGINT NOT NULL REFERENCES book_copies(id),
    borrowed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    due_date DATE NOT NULL,
    returned_at TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'BORROWED',
    fine_amount DECIMAL(8,2) DEFAULT 0,
    fine_paid BOOLEAN DEFAULT FALSE,
    renewal_count INT DEFAULT 0,
    max_renewals INT DEFAULT 2
);

CREATE TABLE book_reservations (
    id BIGSERIAL PRIMARY KEY,
    member_id BIGINT NOT NULL REFERENCES library_members(id),
    book_id BIGINT NOT NULL REFERENCES books(id),
    reserved_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    queue_position INT NOT NULL DEFAULT 1,
    notified_at TIMESTAMP
);

CREATE INDEX idx_borrowing_member ON borrowing_records(member_id, status);
CREATE INDEX idx_borrowing_due_date ON borrowing_records(due_date) WHERE returned_at IS NULL;
CREATE INDEX idx_books_title ON books USING gin(to_tsvector('simple', title));
CREATE INDEX idx_reservations_book ON book_reservations(book_id, status);
```

### Entity

```java
// Book.java
package com.library.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "books")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "isbn", nullable = false, unique = true)
    private String isbn;

    @Column(name = "title", nullable = false)
    private String title;

    @Column(name = "description", columnDefinition = "TEXT")
    private String description;

    @Column(name = "publisher")
    private String publisher;

    @Column(name = "published_year")
    private Integer publishedYear;

    @Column(name = "language")
    private String language = "Thai";

    @Column(name = "pages")
    private Integer pages;

    @Column(name = "category")
    private String category;

    @Column(name = "total_copies")
    private Integer totalCopies = 1;

    @Column(name = "available_copies")
    private Integer availableCopies = 1;

    @Column(name = "cover_image_url")
    private String coverImageUrl;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// LibraryMember.java
@Entity
@Table(name = "library_members")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class LibraryMember {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false, unique = true)
    private Long userId;

    @Column(name = "member_number", nullable = false, unique = true)
    private String memberNumber;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "email", nullable = false)
    private String email;

    @Column(name = "membership_type")
    private String membershipType = "STANDARD";

    @Column(name = "status")
    private String status = "ACTIVE";

    @Column(name = "membership_expiry")
    private java.time.LocalDate membershipExpiry;

    @Column(name = "max_borrow_limit")
    private Integer maxBorrowLimit = 5;

    @Column(name = "current_borrows")
    private Integer currentBorrows = 0;

    @Column(name = "total_fines")
    private java.math.BigDecimal totalFines = java.math.BigDecimal.ZERO;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// BorrowingRecord.java
@Entity
@Table(name = "borrowing_records")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class BorrowingRecord {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "member_id", nullable = false)
    private Long memberId;

    @Column(name = "book_copy_id", nullable = false)
    private Long bookCopyId;

    @Column(name = "borrowed_at", nullable = false)
    private LocalDateTime borrowedAt = LocalDateTime.now();

    @Column(name = "due_date", nullable = false)
    private java.time.LocalDate dueDate;

    @Column(name = "returned_at")
    private LocalDateTime returnedAt;

    @Column(name = "status", nullable = false)
    private String status = "BORROWED";

    @Column(name = "fine_amount")
    private java.math.BigDecimal fineAmount = java.math.BigDecimal.ZERO;

    @Column(name = "fine_paid")
    private Boolean finePaid = false;

    @Column(name = "renewal_count")
    private Integer renewalCount = 0;

    @Column(name = "max_renewals")
    private Integer maxRenewals = 2;
}
```

### Service

```java
// LibraryService.java
package com.library.service;

import lombok.RequiredArgsConstructor;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;

@Service
@RequiredArgsConstructor
public class LibraryService {

    private static final BigDecimal FINE_PER_DAY = BigDecimal.valueOf(5.00);
    private static final int DEFAULT_LOAN_DAYS = 14;

    private final BookRepository bookRepository;
    private final BookCopyRepository copyRepository;
    private final LibraryMemberRepository memberRepository;
    private final BorrowingRecordRepository borrowingRepository;
    private final BookReservationRepository reservationRepository;
    private final NotificationService notificationService;

    @Transactional
    public LibraryMember enrollMember(Long userId, String name, String email, String phone) {
        String memberNumber = "LIB-" + String.format("%06d", System.currentTimeMillis() % 1000000);
        LibraryMember member = LibraryMember.builder()
                .userId(userId).memberNumber(memberNumber).name(name).email(email).phone(phone)
                .membershipExpiry(LocalDate.now().plusYears(1)).build();
        return memberRepository.save(member);
    }

    @Transactional
    public BorrowingRecord borrowBook(Long memberId, Long bookId) {
        LibraryMember member = memberRepository.findById(memberId)
                .orElseThrow(() -> new RuntimeException("Member not found"));
        if (!"ACTIVE".equals(member.getStatus())) {
            throw new IllegalStateException("Member account is not active");
        }
        if (member.getCurrentBorrows() >= member.getMaxBorrowLimit()) {
            throw new IllegalStateException("Borrow limit reached: " + member.getMaxBorrowLimit());
        }
        if (member.getTotalFines().compareTo(BigDecimal.valueOf(100)) > 0) {
            throw new IllegalStateException("Cannot borrow. Outstanding fines exceed 100 THB");
        }

        BookCopy availableCopy = copyRepository
                .findFirstByBookIdAndStatus(bookId, "AVAILABLE")
                .orElseThrow(() -> new RuntimeException("No copies available. Please reserve."));

        availableCopy.setStatus("BORROWED");
        copyRepository.save(availableCopy);

        Book book = bookRepository.findById(bookId).orElseThrow();
        book.setAvailableCopies(Math.max(0, book.getAvailableCopies() - 1));
        bookRepository.save(book);

        member.setCurrentBorrows(member.getCurrentBorrows() + 1);
        memberRepository.save(member);

        BorrowingRecord record = BorrowingRecord.builder()
                .memberId(memberId).bookCopyId(availableCopy.getId())
                .dueDate(LocalDate.now().plusDays(DEFAULT_LOAN_DAYS)).build();
        return borrowingRepository.save(record);
    }

    @Transactional
    public BorrowingRecord returnBook(Long recordId) {
        BorrowingRecord record = borrowingRepository.findById(recordId)
                .orElseThrow(() -> new RuntimeException("Borrowing record not found"));
        if (!"BORROWED".equals(record.getStatus())) {
            throw new IllegalStateException("Book is not currently borrowed");
        }

        record.setReturnedAt(LocalDateTime.now());
        record.setStatus("RETURNED");

        // Calculate fine
        if (record.getDueDate().isBefore(LocalDate.now())) {
            long overdueDays = java.time.temporal.ChronoUnit.DAYS.between(
                record.getDueDate(), LocalDate.now());
            BigDecimal fine = FINE_PER_DAY.multiply(BigDecimal.valueOf(overdueDays));
            record.setFineAmount(fine);

            LibraryMember member = memberRepository.findById(record.getMemberId()).orElseThrow();
            member.setTotalFines(member.getTotalFines().add(fine));
            memberRepository.save(member);
        }

        // Mark copy as available
        BookCopy copy = copyRepository.findById(record.getBookCopyId()).orElseThrow();
        copy.setStatus("AVAILABLE");
        copyRepository.save(copy);

        Book book = bookRepository.findById(copy.getBookId()).orElseThrow();
        book.setAvailableCopies(book.getAvailableCopies() + 1);
        bookRepository.save(book);

        LibraryMember member = memberRepository.findById(record.getMemberId()).orElseThrow();
        member.setCurrentBorrows(Math.max(0, member.getCurrentBorrows() - 1));
        memberRepository.save(member);

        // Fulfill reservations
        fulfillNextReservation(copy.getBookId());
        return borrowingRepository.save(record);
    }

    @Transactional
    public BorrowingRecord renewBook(Long recordId) {
        BorrowingRecord record = borrowingRepository.findById(recordId).orElseThrow();
        if (record.getRenewalCount() >= record.getMaxRenewals()) {
            throw new IllegalStateException("Max renewals reached");
        }
        record.setDueDate(record.getDueDate().plusDays(DEFAULT_LOAN_DAYS));
        record.setRenewalCount(record.getRenewalCount() + 1);
        return borrowingRepository.save(record);
    }

    @Transactional
    public BookReservation reserveBook(Long memberId, Long bookId) {
        int queuePosition = reservationRepository.countByBookIdAndStatus(bookId, "PENDING") + 1;
        BookReservation reservation = BookReservation.builder()
                .memberId(memberId).bookId(bookId)
                .expiresAt(LocalDateTime.now().plusDays(30))
                .queuePosition(queuePosition).build();
        return reservationRepository.save(reservation);
    }

    private void fulfillNextReservation(Long bookId) {
        List<BookReservation> reservations = reservationRepository
                .findByBookIdAndStatusOrderByReservedAt(bookId, "PENDING");
        if (!reservations.isEmpty()) {
            BookReservation next = reservations.get(0);
            next.setStatus("READY");
            next.setNotifiedAt(LocalDateTime.now());
            reservationRepository.save(next);
            LibraryMember member = memberRepository.findById(next.getMemberId()).orElseThrow();
            notificationService.sendBookAvailableNotification(member.getEmail(), bookId);
        }
    }

    @Scheduled(cron = "0 0 8 * * *")
    public void checkOverdueBooks() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        List<BorrowingRecord> overdueRecords = borrowingRepository
                .findByDueDateBeforeAndStatus(yesterday, "BORROWED");
        for (BorrowingRecord record : overdueRecords) {
            record.setStatus("OVERDUE");
            borrowingRepository.save(record);
            LibraryMember member = memberRepository.findById(record.getMemberId()).orElseThrow();
            notificationService.sendOverdueNotification(member.getEmail(), record);
        }
    }

    public List<Book> searchBooks(String query, String category, String language) {
        return bookRepository.searchBooks(query, category, language);
    }
}
```

### Controller

```java
// LibraryController.java
package com.library.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/v1/library")
@RequiredArgsConstructor
public class LibraryController {

    private final LibraryService libraryService;

    @GetMapping("/books")
    public ResponseEntity<List<Book>> searchBooks(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String language) {
        return ResponseEntity.ok(libraryService.searchBooks(q, category, language));
    }

    @PostMapping("/members")
    public ResponseEntity<LibraryMember> enrollMember(@RequestBody EnrollMemberRequest request) {
        return ResponseEntity.ok(libraryService.enrollMember(
            request.userId(), request.name(), request.email(), request.phone()));
    }

    @PostMapping("/borrow")
    public ResponseEntity<BorrowingRecord> borrowBook(@RequestBody BorrowRequest request) {
        return ResponseEntity.ok(libraryService.borrowBook(request.memberId(), request.bookId()));
    }

    @PutMapping("/return/{recordId}")
    public ResponseEntity<BorrowingRecord> returnBook(@PathVariable Long recordId) {
        return ResponseEntity.ok(libraryService.returnBook(recordId));
    }

    @PutMapping("/renew/{recordId}")
    public ResponseEntity<BorrowingRecord> renewBook(@PathVariable Long recordId) {
        return ResponseEntity.ok(libraryService.renewBook(recordId));
    }

    @PostMapping("/reserve")
    public ResponseEntity<BookReservation> reserveBook(@RequestBody ReserveRequest request) {
        return ResponseEntity.ok(libraryService.reserveBook(request.memberId(), request.bookId()));
    }

    record EnrollMemberRequest(Long userId, String name, String email, String phone) {}
    record BorrowRequest(Long memberId, Long bookId) {}
    record ReserveRequest(Long memberId, Long bookId) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Library Management System)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: library_db
      POSTGRES_USER: library_user
      POSTGRES_PASSWORD: library_pass
    ports:
      - "5432:5432"
    volumes:
      - library_pg_data:/var/lib/postgresql/data

  library-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/library_db
      SPRING_DATASOURCE_USERNAME: library_user
      SPRING_DATASOURCE_PASSWORD: library_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  library_pg_data:
```

---

## โปรเจค 67: Language Learning App

### ภาพรวมระบบ

Language Learning App สร้างประสบการณ์การเรียนภาษาที่มีประสิทธิภาพด้วยระบบ Spaced Repetition สำหรับ Flashcards แบบฝึกหัดหลายรูปแบบ ติดตามความก้าวหน้า และสร้าง Streak เพื่อกระตุ้นให้เรียนต่อเนื่อง

- **บทเรียนและคำศัพท์**: จัดการเนื้อหาการเรียน
- **Flashcards (Spaced Repetition)**: ระบบทบทวนคำศัพท์อัจฉริยะ
- **แบบฝึกหัด**: Fill-in-blank และ Multiple Choice
- **Progress Tracking**: ติดตามความก้าวหน้าต่อ Level
- **Streaks**: รางวัลการเรียนต่อเนื่อง

### Flyway Migration

```sql
-- V1__create_language_learning_tables.sql
CREATE TABLE languages (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    native_name VARCHAR(100),
    flag_url VARCHAR(200),
    active BOOLEAN DEFAULT TRUE
);

CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    language_id BIGINT NOT NULL REFERENCES languages(id),
    title VARCHAR(300) NOT NULL,
    description TEXT,
    target_language_id BIGINT NOT NULL REFERENCES languages(id),
    level VARCHAR(20) NOT NULL DEFAULT 'BEGINNER',
    image_url VARCHAR(500),
    total_lessons INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE lessons (
    id BIGSERIAL PRIMARY KEY,
    course_id BIGINT NOT NULL REFERENCES courses(id),
    title VARCHAR(300) NOT NULL,
    description TEXT,
    lesson_order INT NOT NULL,
    xp_reward INT DEFAULT 10,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE vocabulary (
    id BIGSERIAL PRIMARY KEY,
    lesson_id BIGINT REFERENCES lessons(id),
    course_id BIGINT NOT NULL REFERENCES courses(id),
    word VARCHAR(300) NOT NULL,
    translation VARCHAR(300) NOT NULL,
    pronunciation VARCHAR(300),
    example_sentence TEXT,
    example_translation TEXT,
    audio_url VARCHAR(500),
    image_url VARCHAR(500),
    part_of_speech VARCHAR(50),
    difficulty INT DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE user_progress (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL REFERENCES courses(id),
    current_lesson_id BIGINT REFERENCES lessons(id),
    completed_lessons INT DEFAULT 0,
    total_xp INT DEFAULT 0,
    current_streak INT DEFAULT 0,
    longest_streak INT DEFAULT 0,
    last_study_date DATE,
    started_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, course_id)
);

CREATE TABLE flashcard_reviews (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    vocabulary_id BIGINT NOT NULL REFERENCES vocabulary(id),
    ease_factor DECIMAL(4,2) NOT NULL DEFAULT 2.5,
    interval_days INT NOT NULL DEFAULT 1,
    repetitions INT NOT NULL DEFAULT 0,
    next_review_date DATE NOT NULL DEFAULT CURRENT_DATE,
    last_quality INT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, vocabulary_id)
);

CREATE TABLE exercise_submissions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    lesson_id BIGINT NOT NULL REFERENCES lessons(id),
    exercise_type VARCHAR(30) NOT NULL,
    question TEXT NOT NULL,
    user_answer TEXT,
    correct_answer TEXT NOT NULL,
    is_correct BOOLEAN NOT NULL,
    time_taken_seconds INT,
    submitted_at TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO languages (code, name, native_name) VALUES
('EN', 'English', 'English'),
('TH', 'Thai', 'ภาษาไทย'),
('JP', 'Japanese', '日本語'),
('ZH', 'Chinese', '中文'),
('KO', 'Korean', '한국어');
```

### Entity & Service

```java
// LanguageLearningService.java
package com.languagelearning.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;

@Service
@RequiredArgsConstructor
public class LanguageLearningService {

    private final UserProgressRepository progressRepository;
    private final FlashcardReviewRepository reviewRepository;
    private final VocabularyRepository vocabularyRepository;
    private final ExerciseSubmissionRepository submissionRepository;
    private final LessonRepository lessonRepository;

    @Transactional
    public UserProgress enrollCourse(Long userId, Long courseId) {
        return progressRepository.findByUserIdAndCourseId(userId, courseId)
                .orElseGet(() -> {
                    UserProgress progress = UserProgress.builder()
                            .userId(userId).courseId(courseId).build();
                    return progressRepository.save(progress);
                });
    }

    @Transactional
    public FlashcardReview reviewFlashcard(Long userId, Long vocabularyId, int quality) {
        // quality: 0-5 (0=blackout, 5=perfect)
        FlashcardReview review = reviewRepository.findByUserIdAndVocabularyId(userId, vocabularyId)
                .orElse(FlashcardReview.builder()
                    .userId(userId).vocabularyId(vocabularyId)
                    .easeFactor(BigDecimal.valueOf(2.5)).intervalDays(1).repetitions(0).build());

        // SM-2 Algorithm
        double ef = review.getEaseFactor().doubleValue();
        int interval;
        int reps = review.getRepetitions();

        if (quality >= 3) {
            if (reps == 0) interval = 1;
            else if (reps == 1) interval = 6;
            else interval = (int) Math.round(review.getIntervalDays() * ef);
            reps++;
            ef = ef + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02));
            ef = Math.max(1.3, ef);
        } else {
            reps = 0;
            interval = 1;
        }

        review.setEaseFactor(BigDecimal.valueOf(ef));
        review.setIntervalDays(interval);
        review.setRepetitions(reps);
        review.setLastQuality(quality);
        review.setNextReviewDate(LocalDate.now().plusDays(interval));
        review.setUpdatedAt(LocalDateTime.now());
        return reviewRepository.save(review);
    }

    public List<Vocabulary> getDueFlashcards(Long userId, Long courseId, int limit) {
        List<FlashcardReview> dueReviews = reviewRepository
                .findByUserIdAndNextReviewDateLessThanEqualOrderByNextReviewDate(
                    userId, LocalDate.now(), limit);
        if (dueReviews.size() < limit) {
            // Add new cards
            List<Long> reviewedIds = dueReviews.stream().map(FlashcardReview::getVocabularyId).toList();
            List<Vocabulary> newCards = vocabularyRepository
                    .findByCourseIdAndIdNotInOrderByDifficulty(courseId, reviewedIds,
                        limit - dueReviews.size());
            List<Vocabulary> dueCards = dueReviews.stream()
                    .map(r -> vocabularyRepository.findById(r.getVocabularyId()).orElseThrow())
                    .toList();
            List<Vocabulary> result = new java.util.ArrayList<>(dueCards);
            result.addAll(newCards);
            return result;
        }
        return dueReviews.stream()
                .map(r -> vocabularyRepository.findById(r.getVocabularyId()).orElseThrow())
                .toList();
    }

    @Transactional
    public ExerciseSubmission submitExercise(Long userId, Long lessonId, String exerciseType,
                                              String question, String userAnswer,
                                              String correctAnswer, int timeTaken) {
        boolean isCorrect = correctAnswer.trim().equalsIgnoreCase(userAnswer.trim());
        ExerciseSubmission submission = ExerciseSubmission.builder()
                .userId(userId).lessonId(lessonId).exerciseType(exerciseType)
                .question(question).userAnswer(userAnswer).correctAnswer(correctAnswer)
                .isCorrect(isCorrect).timeTakenSeconds(timeTaken).build();
        submission = submissionRepository.save(submission);

        if (isCorrect) {
            updateStreak(userId);
        }
        return submission;
    }

    private void updateStreak(Long userId) {
        // Update streak across all courses for this user - simplified
        List<UserProgress> allProgress = progressRepository.findByUserId(userId);
        for (UserProgress p : allProgress) {
            LocalDate today = LocalDate.now();
            if (p.getLastStudyDate() == null || p.getLastStudyDate().isBefore(today)) {
                if (p.getLastStudyDate() != null && p.getLastStudyDate().equals(today.minusDays(1))) {
                    p.setCurrentStreak(p.getCurrentStreak() + 1);
                } else if (p.getLastStudyDate() == null ||
                           p.getLastStudyDate().isBefore(today.minusDays(1))) {
                    p.setCurrentStreak(1);
                }
                if (p.getCurrentStreak() > p.getLongestStreak()) {
                    p.setLongestStreak(p.getCurrentStreak());
                }
                p.setLastStudyDate(today);
                progressRepository.save(p);
            }
        }
    }

    public LearningStats getStats(Long userId, Long courseId) {
        UserProgress progress = progressRepository.findByUserIdAndCourseId(userId, courseId)
                .orElseThrow(() -> new RuntimeException("Not enrolled in course"));
        long dueCards = reviewRepository.countByUserIdAndNextReviewDateLessThanEqual(
            userId, LocalDate.now());
        return new LearningStats(progress.getCurrentStreak(), progress.getLongestStreak(),
            progress.getTotalXp(), progress.getCompletedLessons(), (int) dueCards);
    }

    public record LearningStats(int currentStreak, int longestStreak, int totalXp,
                                 int completedLessons, int cardsToReview) {}
}
```

### Controller

```java
// LanguageLearningController.java
@RestController
@RequestMapping("/api/v1/language")
@RequiredArgsConstructor
public class LanguageLearningController {

    private final LanguageLearningService service;

    @PostMapping("/enroll")
    public ResponseEntity<UserProgress> enroll(@RequestParam Long userId,
            @RequestParam Long courseId) {
        return ResponseEntity.ok(service.enrollCourse(userId, courseId));
    }

    @GetMapping("/flashcards")
    public ResponseEntity<List<Vocabulary>> getFlashcards(@RequestParam Long userId,
            @RequestParam Long courseId, @RequestParam(defaultValue = "20") int limit) {
        return ResponseEntity.ok(service.getDueFlashcards(userId, courseId, limit));
    }

    @PostMapping("/flashcards/{vocabId}/review")
    public ResponseEntity<FlashcardReview> reviewCard(@PathVariable Long vocabId,
            @RequestParam Long userId, @RequestParam int quality) {
        return ResponseEntity.ok(service.reviewFlashcard(userId, vocabId, quality));
    }

    @PostMapping("/exercises/submit")
    public ResponseEntity<ExerciseSubmission> submitExercise(@RequestBody SubmitExerciseRequest req) {
        return ResponseEntity.ok(service.submitExercise(req.userId(), req.lessonId(),
            req.exerciseType(), req.question(), req.userAnswer(), req.correctAnswer(), req.timeTaken()));
    }

    @GetMapping("/stats/{userId}/{courseId}")
    public ResponseEntity<LanguageLearningService.LearningStats> getStats(
            @PathVariable Long userId, @PathVariable Long courseId) {
        return ResponseEntity.ok(service.getStats(userId, courseId));
    }

    record SubmitExerciseRequest(Long userId, Long lessonId, String exerciseType,
        String question, String userAnswer, String correctAnswer, int timeTaken) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Language Learning App)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: langlearn_db
      POSTGRES_USER: lang_user
      POSTGRES_PASSWORD: lang_pass
    ports:
      - "5432:5432"
    volumes:
      - lang_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  langlearn-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/langlearn_db
      SPRING_DATASOURCE_USERNAME: lang_user
      SPRING_DATASOURCE_PASSWORD: lang_pass
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  lang_pg_data:
```

---

## โปรเจค 68: Quiz & Assessment Platform

### ภาพรวมระบบ

Quiz & Assessment Platform สำหรับสร้างแบบทดสอบ จัดสอบออนไลน์ พร้อม Auto-grading และออกใบประกาศนียบัตร ระบบมีฟีเจอร์ป้องกันการโกงโดยตรวจจับการเปลี่ยน Tab และ API สำหรับวิเคราะห์คำถาม

### Flyway Migration

```sql
-- V1__create_quiz_tables.sql
CREATE TABLE question_banks (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(300) NOT NULL,
    subject VARCHAR(100),
    created_by BIGINT NOT NULL,
    is_public BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE questions (
    id BIGSERIAL PRIMARY KEY,
    bank_id BIGINT NOT NULL REFERENCES question_banks(id),
    question_text TEXT NOT NULL,
    question_type VARCHAR(30) NOT NULL DEFAULT 'MULTIPLE_CHOICE',
    correct_answer TEXT NOT NULL,
    explanation TEXT,
    difficulty INT DEFAULT 3 CHECK (difficulty BETWEEN 1 AND 5),
    points DECIMAL(6,2) DEFAULT 1.0,
    time_limit_seconds INT,
    times_answered INT DEFAULT 0,
    times_correct INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE question_options (
    id BIGSERIAL PRIMARY KEY,
    question_id BIGINT NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    option_key VARCHAR(5) NOT NULL,
    option_text TEXT NOT NULL,
    is_correct BOOLEAN NOT NULL DEFAULT FALSE,
    display_order INT DEFAULT 0
);

CREATE TABLE quizzes (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    description TEXT,
    created_by BIGINT NOT NULL,
    time_limit_minutes INT,
    passing_score DECIMAL(5,2) DEFAULT 60.0,
    max_attempts INT DEFAULT 1,
    randomize_questions BOOLEAN DEFAULT FALSE,
    randomize_options BOOLEAN DEFAULT FALSE,
    show_answers_after BOOLEAN DEFAULT TRUE,
    is_published BOOLEAN DEFAULT FALSE,
    certificate_enabled BOOLEAN DEFAULT FALSE,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE quiz_questions (
    quiz_id BIGINT NOT NULL REFERENCES quizzes(id),
    question_id BIGINT NOT NULL REFERENCES questions(id),
    display_order INT DEFAULT 0,
    PRIMARY KEY (quiz_id, question_id)
);

CREATE TABLE quiz_attempts (
    id BIGSERIAL PRIMARY KEY,
    quiz_id BIGINT NOT NULL REFERENCES quizzes(id),
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'IN_PROGRESS',
    started_at TIMESTAMP NOT NULL DEFAULT NOW(),
    submitted_at TIMESTAMP,
    time_taken_seconds INT,
    score DECIMAL(6,2),
    max_score DECIMAL(6,2),
    percentage DECIMAL(5,2),
    passed BOOLEAN,
    tab_switch_count INT DEFAULT 0,
    certificate_id VARCHAR(100)
);

CREATE TABLE quiz_answers (
    id BIGSERIAL PRIMARY KEY,
    attempt_id BIGINT NOT NULL REFERENCES quiz_attempts(id),
    question_id BIGINT NOT NULL REFERENCES questions(id),
    user_answer TEXT,
    is_correct BOOLEAN,
    points_earned DECIMAL(6,2) DEFAULT 0,
    time_taken_seconds INT,
    answered_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_quiz_attempts_user ON quiz_attempts(user_id, quiz_id);
CREATE INDEX idx_quiz_answers_attempt ON quiz_answers(attempt_id);
```

### Entity & Service

```java
// QuizService.java
package com.quiz.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class QuizService {

    private final QuizRepository quizRepository;
    private final QuizAttemptRepository attemptRepository;
    private final QuizAnswerRepository answerRepository;
    private final QuestionRepository questionRepository;
    private final CertificateService certificateService;

    @Transactional
    public QuizAttempt startAttempt(Long userId, Long quizId) {
        Quiz quiz = quizRepository.findById(quizId)
                .orElseThrow(() -> new RuntimeException("Quiz not found"));
        if (!quiz.getIsPublished()) {
            throw new IllegalStateException("Quiz is not published");
        }

        long existingAttempts = attemptRepository.countByUserIdAndQuizId(userId, quizId);
        if (quiz.getMaxAttempts() != null && existingAttempts >= quiz.getMaxAttempts()) {
            throw new IllegalStateException("Max attempts reached: " + quiz.getMaxAttempts());
        }

        if (quiz.getStartTime() != null && quiz.getStartTime().isAfter(LocalDateTime.now())) {
            throw new IllegalStateException("Quiz has not started yet");
        }
        if (quiz.getEndTime() != null && quiz.getEndTime().isBefore(LocalDateTime.now())) {
            throw new IllegalStateException("Quiz has ended");
        }

        QuizAttempt attempt = QuizAttempt.builder()
                .quizId(quizId).userId(userId).status("IN_PROGRESS").build();
        return attemptRepository.save(attempt);
    }

    @Transactional
    public QuizAnswer submitAnswer(Long attemptId, Long questionId, String userAnswer) {
        QuizAttempt attempt = attemptRepository.findById(attemptId)
                .orElseThrow(() -> new RuntimeException("Attempt not found"));
        if (!"IN_PROGRESS".equals(attempt.getStatus())) {
            throw new IllegalStateException("Attempt is not in progress");
        }

        Question question = questionRepository.findById(questionId).orElseThrow();
        boolean isCorrect = question.getCorrectAnswer().trim()
                .equalsIgnoreCase(userAnswer != null ? userAnswer.trim() : "");
        BigDecimal pointsEarned = isCorrect ? question.getPoints() : BigDecimal.ZERO;

        // Update question analytics
        question.setTimesAnswered(question.getTimesAnswered() + 1);
        if (isCorrect) question.setTimesCorrect(question.getTimesCorrect() + 1);
        questionRepository.save(question);

        QuizAnswer answer = QuizAnswer.builder()
                .attemptId(attemptId).questionId(questionId)
                .userAnswer(userAnswer).isCorrect(isCorrect)
                .pointsEarned(pointsEarned).build();
        return answerRepository.save(answer);
    }

    @Transactional
    public QuizAttempt submitAttempt(Long attemptId) {
        QuizAttempt attempt = attemptRepository.findById(attemptId)
                .orElseThrow(() -> new RuntimeException("Attempt not found"));
        Quiz quiz = quizRepository.findById(attempt.getQuizId()).orElseThrow();

        List<QuizAnswer> answers = answerRepository.findByAttemptId(attemptId);
        BigDecimal totalScore = answers.stream().map(a -> a.getPointsEarned() != null ?
                a.getPointsEarned() : BigDecimal.ZERO).reduce(BigDecimal.ZERO, BigDecimal::add);

        List<Question> questions = questionRepository.findByQuizId(attempt.getQuizId());
        BigDecimal maxScore = questions.stream().map(Question::getPoints)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal percentage = maxScore.compareTo(BigDecimal.ZERO) > 0 ?
            totalScore.divide(maxScore, 4, RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(100)) : BigDecimal.ZERO;

        boolean passed = percentage.compareTo(quiz.getPassingScore()) >= 0;
        long timeTaken = java.time.Duration.between(attempt.getStartedAt(), LocalDateTime.now()).getSeconds();

        attempt.setStatus("COMPLETED");
        attempt.setSubmittedAt(LocalDateTime.now());
        attempt.setTimeTakenSeconds((int) timeTaken);
        attempt.setScore(totalScore);
        attempt.setMaxScore(maxScore);
        attempt.setPercentage(percentage);
        attempt.setPassed(passed);

        if (passed && quiz.getCertificateEnabled()) {
            String certId = certificateService.issueCertificate(
                attempt.getUserId(), quiz.getId(), percentage);
            attempt.setCertificateId(certId);
        }

        return attemptRepository.save(attempt);
    }

    @Transactional
    public void recordTabSwitch(Long attemptId) {
        QuizAttempt attempt = attemptRepository.findById(attemptId).orElseThrow();
        attempt.setTabSwitchCount(attempt.getTabSwitchCount() + 1);
        // Auto-submit if too many tab switches
        if (attempt.getTabSwitchCount() >= 3) {
            attempt.setStatus("AUTO_SUBMITTED_CHEATING");
            submitAttempt(attemptId);
        } else {
            attemptRepository.save(attempt);
        }
    }

    public QuestionAnalytics getQuestionAnalytics(Long questionId) {
        Question q = questionRepository.findById(questionId).orElseThrow();
        double accuracy = q.getTimesAnswered() > 0 ?
            (double) q.getTimesCorrect() / q.getTimesAnswered() * 100 : 0.0;
        return new QuestionAnalytics(questionId, q.getTimesAnswered(), q.getTimesCorrect(),
            accuracy, q.getDifficulty());
    }

    public record QuestionAnalytics(Long questionId, int timesAnswered, int timesCorrect,
                                     double accuracyRate, int difficulty) {}
}
```

### Controller & docker-compose.yml

```java
// QuizController.java
@RestController
@RequestMapping("/api/v1/quiz")
@RequiredArgsConstructor
public class QuizController {

    private final QuizService quizService;

    @PostMapping("/{quizId}/start")
    public ResponseEntity<QuizAttempt> start(@PathVariable Long quizId,
            @RequestParam Long userId) {
        return ResponseEntity.ok(quizService.startAttempt(userId, quizId));
    }

    @PostMapping("/attempts/{attemptId}/answers")
    public ResponseEntity<QuizAnswer> submitAnswer(@PathVariable Long attemptId,
            @RequestBody SubmitAnswerRequest request) {
        return ResponseEntity.ok(quizService.submitAnswer(
            attemptId, request.questionId(), request.answer()));
    }

    @PostMapping("/attempts/{attemptId}/submit")
    public ResponseEntity<QuizAttempt> submit(@PathVariable Long attemptId) {
        return ResponseEntity.ok(quizService.submitAttempt(attemptId));
    }

    @PostMapping("/attempts/{attemptId}/tab-switch")
    public ResponseEntity<Void> recordTabSwitch(@PathVariable Long attemptId) {
        quizService.recordTabSwitch(attemptId);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/questions/{questionId}/analytics")
    public ResponseEntity<QuizService.QuestionAnalytics> getAnalytics(
            @PathVariable Long questionId) {
        return ResponseEntity.ok(quizService.getQuestionAnalytics(questionId));
    }

    record SubmitAnswerRequest(Long questionId, String answer) {}
}
```

```yaml
# docker-compose.yml (Quiz & Assessment Platform)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: quiz_db
      POSTGRES_USER: quiz_user
      POSTGRES_PASSWORD: quiz_pass
    ports:
      - "5432:5432"
    volumes:
      - quiz_pg_data:/var/lib/postgresql/data

  quiz-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/quiz_db
      SPRING_DATASOURCE_USERNAME: quiz_user
      SPRING_DATASOURCE_PASSWORD: quiz_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  quiz_pg_data:
```

---

## โปรเจค 69: Online Code Challenge Platform

### ภาพรวมระบบ

Online Code Challenge Platform คล้าย LeetCode หรือ HackerRank ผู้ใช้สามารถส่ง Code เพื่อแก้ปัญหา ระบบจะ Execute Code และเปรียบเทียบ Output กับ Test Cases บันทึก Leaderboard และจัดระดับความยาก

### Flyway Migration

```sql
-- V1__create_code_challenge_tables.sql
CREATE TABLE challenges (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    slug VARCHAR(300) NOT NULL UNIQUE,
    description TEXT NOT NULL,
    difficulty VARCHAR(20) NOT NULL DEFAULT 'EASY',
    category VARCHAR(100),
    time_limit_ms INT NOT NULL DEFAULT 2000,
    memory_limit_mb INT NOT NULL DEFAULT 256,
    total_submissions INT DEFAULT 0,
    accepted_submissions INT DEFAULT 0,
    created_by BIGINT,
    is_published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE test_cases (
    id BIGSERIAL PRIMARY KEY,
    challenge_id BIGINT NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
    input TEXT NOT NULL,
    expected_output TEXT NOT NULL,
    is_sample BOOLEAN DEFAULT FALSE,
    points DECIMAL(6,2) DEFAULT 10.0,
    display_order INT DEFAULT 0
);

CREATE TABLE code_submissions (
    id BIGSERIAL PRIMARY KEY,
    challenge_id BIGINT NOT NULL REFERENCES challenges(id),
    user_id BIGINT NOT NULL,
    language VARCHAR(30) NOT NULL,
    code TEXT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    verdict VARCHAR(30),
    total_test_cases INT,
    passed_test_cases INT,
    score DECIMAL(8,2) DEFAULT 0,
    execution_time_ms INT,
    memory_used_mb INT,
    error_message TEXT,
    submitted_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE submission_results (
    id BIGSERIAL PRIMARY KEY,
    submission_id BIGINT NOT NULL REFERENCES code_submissions(id),
    test_case_id BIGINT NOT NULL REFERENCES test_cases(id),
    status VARCHAR(20) NOT NULL,
    actual_output TEXT,
    execution_time_ms INT,
    memory_used_mb INT,
    error_message TEXT
);

CREATE TABLE challenge_leaderboard (
    id BIGSERIAL PRIMARY KEY,
    challenge_id BIGINT NOT NULL REFERENCES challenges(id),
    user_id BIGINT NOT NULL,
    best_score DECIMAL(8,2) NOT NULL,
    best_time_ms INT,
    language VARCHAR(30),
    submission_id BIGINT REFERENCES code_submissions(id),
    rank INT,
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (challenge_id, user_id)
);

CREATE INDEX idx_submissions_challenge_user ON code_submissions(challenge_id, user_id);
CREATE INDEX idx_leaderboard_challenge ON challenge_leaderboard(challenge_id, best_score DESC);
```

### Entity & Service

```java
// CodeChallengeService.java
package com.codechallenge.service;

import lombok.RequiredArgsConstructor;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.util.*;

@Service
@RequiredArgsConstructor
public class CodeChallengeService {

    private final ChallengeRepository challengeRepository;
    private final CodeSubmissionRepository submissionRepository;
    private final SubmissionResultRepository resultRepository;
    private final TestCaseRepository testCaseRepository;
    private final LeaderboardRepository leaderboardRepository;
    private final CodeExecutionService executionService;

    @Transactional
    public CodeSubmission submit(Long userId, Long challengeId, String language, String code) {
        Challenge challenge = challengeRepository.findById(challengeId)
                .orElseThrow(() -> new RuntimeException("Challenge not found"));
        if (!challenge.getIsPublished()) {
            throw new IllegalStateException("Challenge is not published");
        }

        CodeSubmission submission = CodeSubmission.builder()
                .challengeId(challengeId).userId(userId)
                .language(language).code(code).status("PENDING").build();
        submission = submissionRepository.save(submission);

        challenge.setTotalSubmissions(challenge.getTotalSubmissions() + 1);
        challengeRepository.save(challenge);

        // Async execution
        executeSubmission(submission.getId(), challenge);
        return submission;
    }

    @Async
    @Transactional
    public void executeSubmission(Long submissionId, Challenge challenge) {
        CodeSubmission submission = submissionRepository.findById(submissionId).orElseThrow();
        List<TestCase> testCases = testCaseRepository.findByChallengeIdOrderByDisplayOrder(
            challenge.getId());

        submission.setStatus("RUNNING");
        submissionRepository.save(submission);

        int passed = 0;
        BigDecimal totalScore = BigDecimal.ZERO;
        int maxTime = 0;
        String verdict = "ACCEPTED";

        for (TestCase testCase : testCases) {
            ExecutionResult result = executionService.execute(
                submission.getCode(), submission.getLanguage(),
                testCase.getInput(), challenge.getTimeLimitMs(), challenge.getMemoryLimitMb());

            SubmissionResult sr = SubmissionResult.builder()
                    .submissionId(submissionId).testCaseId(testCase.getId())
                    .status(result.status()).actualOutput(result.output())
                    .executionTimeMs(result.timeMs()).memoryUsedMb(result.memoryMb())
                    .errorMessage(result.error()).build();
            resultRepository.save(sr);

            if ("ACCEPTED".equals(result.status())) {
                passed++;
                totalScore = totalScore.add(testCase.getPoints());
                maxTime = Math.max(maxTime, result.timeMs() != null ? result.timeMs() : 0);
            } else {
                if (verdict.equals("ACCEPTED")) verdict = result.status();
            }
        }

        submission.setStatus("COMPLETED");
        submission.setVerdict(verdict);
        submission.setTotalTestCases(testCases.size());
        submission.setPassedTestCases(passed);
        submission.setScore(totalScore);
        submission.setExecutionTimeMs(maxTime);
        submissionRepository.save(submission);

        if ("ACCEPTED".equals(verdict)) {
            challenge.setAcceptedSubmissions(challenge.getAcceptedSubmissions() + 1);
            challengeRepository.save(challenge);
            updateLeaderboard(submission, totalScore, maxTime, challenge.getId());
        }
    }

    private void updateLeaderboard(CodeSubmission submission, BigDecimal score,
                                    int timeMs, Long challengeId) {
        ChallengeLeaderboard entry = leaderboardRepository
                .findByChallengeIdAndUserId(challengeId, submission.getUserId())
                .orElse(ChallengeLeaderboard.builder()
                    .challengeId(challengeId).userId(submission.getUserId())
                    .bestScore(BigDecimal.ZERO).build());

        if (score.compareTo(entry.getBestScore()) > 0 ||
            (score.compareTo(entry.getBestScore()) == 0 &&
             (entry.getBestTimeMs() == null || timeMs < entry.getBestTimeMs()))) {
            entry.setBestScore(score);
            entry.setBestTimeMs(timeMs);
            entry.setLanguage(submission.getLanguage());
            entry.setSubmissionId(submission.getId());
            entry.setUpdatedAt(java.time.LocalDateTime.now());
            leaderboardRepository.save(entry);
            recalculateRanks(challengeId);
        }
    }

    private void recalculateRanks(Long challengeId) {
        List<ChallengeLeaderboard> entries = leaderboardRepository
                .findByChallengeIdOrderByBestScoreDescBestTimeMsAsc(challengeId);
        for (int i = 0; i < entries.size(); i++) {
            entries.get(i).setRank(i + 1);
        }
        leaderboardRepository.saveAll(entries);
    }

    public List<ChallengeLeaderboard> getLeaderboard(Long challengeId, int limit) {
        return leaderboardRepository
                .findByChallengeIdOrderByRankAsc(challengeId, limit);
    }

    public record ExecutionResult(String status, String output, Integer timeMs,
                                   Integer memoryMb, String error) {}
}
```

### Controller & docker-compose.yml

```java
// CodeChallengeController.java
@RestController
@RequestMapping("/api/v1/challenges")
@RequiredArgsConstructor
public class CodeChallengeController {

    private final CodeChallengeService service;
    private final ChallengeRepository challengeRepository;

    @GetMapping
    public ResponseEntity<List<Challenge>> getChallenges(
            @RequestParam(required = false) String difficulty,
            @RequestParam(required = false) String category) {
        return ResponseEntity.ok(challengeRepository.findByFilters(difficulty, category, true));
    }

    @GetMapping("/{slug}")
    public ResponseEntity<Challenge> getBySlug(@PathVariable String slug) {
        return ResponseEntity.ok(challengeRepository.findBySlugAndIsPublishedTrue(slug)
                .orElseThrow(() -> new RuntimeException("Challenge not found")));
    }

    @PostMapping("/{challengeId}/submit")
    public ResponseEntity<CodeSubmission> submit(@PathVariable Long challengeId,
            @RequestBody SubmitCodeRequest request) {
        return ResponseEntity.ok(service.submit(request.userId(), challengeId,
            request.language(), request.code()));
    }

    @GetMapping("/{challengeId}/leaderboard")
    public ResponseEntity<List<ChallengeLeaderboard>> getLeaderboard(
            @PathVariable Long challengeId,
            @RequestParam(defaultValue = "20") int limit) {
        return ResponseEntity.ok(service.getLeaderboard(challengeId, limit));
    }

    record SubmitCodeRequest(Long userId, String language, String code) {}
}
```

```yaml
# docker-compose.yml (Online Code Challenge Platform)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: codechallenge_db
      POSTGRES_USER: code_user
      POSTGRES_PASSWORD: code_pass
    ports:
      - "5432:5432"
    volumes:
      - code_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  code-executor:
    image: code-executor:latest
    environment:
      MAX_EXECUTION_TIME: 5000
      MAX_MEMORY_MB: 512
    ports:
      - "8081:8081"

  codechallenge-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/codechallenge_db
      SPRING_DATASOURCE_USERNAME: code_user
      SPRING_DATASOURCE_PASSWORD: code_pass
      SPRING_REDIS_HOST: redis
      CODE_EXECUTOR_URL: http://code-executor:8081
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - code-executor

volumes:
  code_pg_data:
```

---

## โปรเจค 70: Book Club Platform

### ภาพรวมระบบ

Book Club Platform เชื่อมต่อนักอ่านที่มีความสนใจเหมือนกัน จัดการกลุ่มอ่านหนังสือ ตารางการอ่าน การอภิปราย ติดตามความก้าวหน้าการอ่าน ให้คะแนน และระบบแนะนำหนังสือเล่มถัดไป

### Flyway Migration

```sql
-- V1__create_book_club_tables.sql
CREATE TABLE book_clubs (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(300) NOT NULL,
    description TEXT,
    genre_focus VARCHAR(100),
    image_url VARCHAR(500),
    owner_id BIGINT NOT NULL,
    is_private BOOLEAN DEFAULT FALSE,
    max_members INT DEFAULT 50,
    current_members INT DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE club_members (
    id BIGSERIAL PRIMARY KEY,
    club_id BIGINT NOT NULL REFERENCES book_clubs(id),
    user_id BIGINT NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'MEMBER',
    joined_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (club_id, user_id)
);

CREATE TABLE club_books (
    id BIGSERIAL PRIMARY KEY,
    club_id BIGINT NOT NULL REFERENCES book_clubs(id),
    title VARCHAR(500) NOT NULL,
    author VARCHAR(300) NOT NULL,
    isbn VARCHAR(20),
    cover_url VARCHAR(500),
    total_pages INT,
    description TEXT,
    is_monthly_pick BOOLEAN DEFAULT FALSE,
    reading_start_date DATE,
    reading_end_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'UPCOMING',
    added_by BIGINT NOT NULL,
    added_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE reading_progress (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    club_book_id BIGINT NOT NULL REFERENCES club_books(id),
    pages_read INT NOT NULL DEFAULT 0,
    percentage_complete DECIMAL(5,2) DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'NOT_STARTED',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, club_book_id)
);

CREATE TABLE discussions (
    id BIGSERIAL PRIMARY KEY,
    club_id BIGINT NOT NULL REFERENCES book_clubs(id),
    club_book_id BIGINT REFERENCES club_books(id),
    user_id BIGINT NOT NULL,
    title VARCHAR(300),
    content TEXT NOT NULL,
    spoiler_warning BOOLEAN DEFAULT FALSE,
    parent_id BIGINT REFERENCES discussions(id),
    like_count INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE book_ratings (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    club_book_id BIGINT NOT NULL REFERENCES club_books(id),
    rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, club_book_id)
);
```

### Entity & Service

```java
// BookClubService.java
package com.bookclub.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;

@Service
@RequiredArgsConstructor
public class BookClubService {

    private final BookClubRepository clubRepository;
    private final ClubMemberRepository memberRepository;
    private final ClubBookRepository bookRepository;
    private final ReadingProgressRepository progressRepository;
    private final DiscussionRepository discussionRepository;
    private final BookRatingRepository ratingRepository;

    @Transactional
    public BookClub createClub(Long ownerId, String name, String description,
                                String genreFocus, boolean isPrivate) {
        BookClub club = BookClub.builder()
                .name(name).description(description).genreFocus(genreFocus)
                .ownerId(ownerId).isPrivate(isPrivate).currentMembers(1).build();
        club = clubRepository.save(club);

        // Owner is first member
        ClubMember ownerMember = ClubMember.builder()
                .clubId(club.getId()).userId(ownerId).role("OWNER").build();
        memberRepository.save(ownerMember);
        return club;
    }

    @Transactional
    public ClubMember joinClub(Long clubId, Long userId) {
        BookClub club = clubRepository.findById(clubId)
                .orElseThrow(() -> new RuntimeException("Club not found"));
        if (memberRepository.existsByClubIdAndUserId(clubId, userId)) {
            throw new IllegalStateException("Already a member of this club");
        }
        if (club.getCurrentMembers() >= club.getMaxMembers()) {
            throw new IllegalStateException("Club is full");
        }

        ClubMember member = ClubMember.builder()
                .clubId(clubId).userId(userId).role("MEMBER").build();
        member = memberRepository.save(member);

        club.setCurrentMembers(club.getCurrentMembers() + 1);
        clubRepository.save(club);
        return member;
    }

    @Transactional
    public ClubBook addBook(Long clubId, Long addedBy, String title, String author,
                             String isbn, int totalPages, boolean isMonthlyPick,
                             LocalDate startDate, LocalDate endDate) {
        ClubBook book = ClubBook.builder()
                .clubId(clubId).title(title).author(author).isbn(isbn)
                .totalPages(totalPages).isMonthlyPick(isMonthlyPick)
                .readingStartDate(startDate).readingEndDate(endDate)
                .status("UPCOMING").addedBy(addedBy).build();
        return bookRepository.save(book);
    }

    @Transactional
    public ReadingProgress updateProgress(Long userId, Long clubBookId, int pagesRead) {
        ClubBook book = bookRepository.findById(clubBookId).orElseThrow();
        ReadingProgress progress = progressRepository.findByUserIdAndClubBookId(userId, clubBookId)
                .orElse(ReadingProgress.builder()
                    .userId(userId).clubBookId(clubBookId)
                    .status("IN_PROGRESS").startedAt(LocalDateTime.now()).build());

        progress.setPagesRead(pagesRead);
        if (book.getTotalPages() != null && book.getTotalPages() > 0) {
            double pct = (double) pagesRead / book.getTotalPages() * 100;
            progress.setPercentageComplete(BigDecimal.valueOf(Math.min(100.0, pct))
                .setScale(2, java.math.RoundingMode.HALF_UP));
        }
        if (book.getTotalPages() != null && pagesRead >= book.getTotalPages()) {
            progress.setStatus("COMPLETED");
            progress.setCompletedAt(LocalDateTime.now());
        } else {
            progress.setStatus("IN_PROGRESS");
        }
        progress.setUpdatedAt(LocalDateTime.now());
        return progressRepository.save(progress);
    }

    @Transactional
    public Discussion postDiscussion(Long clubId, Long userId, Long clubBookId,
                                      String title, String content, boolean spoilerWarning,
                                      Long parentId) {
        Discussion discussion = Discussion.builder()
                .clubId(clubId).clubBookId(clubBookId).userId(userId)
                .title(title).content(content).spoilerWarning(spoilerWarning)
                .parentId(parentId).build();
        return discussionRepository.save(discussion);
    }

    @Transactional
    public BookRating rateBook(Long userId, Long clubBookId, int rating, String review) {
        BookRating bookRating = ratingRepository.findByUserIdAndClubBookId(userId, clubBookId)
                .orElse(BookRating.builder().userId(userId).clubBookId(clubBookId).build());
        bookRating.setRating(rating);
        bookRating.setReview(review);
        return ratingRepository.save(bookRating);
    }

    public ClubBook selectMonthlyPick(Long clubId) {
        // Recommend based on ratings and genre
        List<ClubBook> upcoming = bookRepository.findByClubIdAndStatus(clubId, "UPCOMING");
        if (upcoming.isEmpty()) throw new RuntimeException("No upcoming books to select from");
        // Select highest rated or first in queue
        return upcoming.stream()
                .filter(b -> !b.getIsMonthlyPick())
                .findFirst()
                .orElse(upcoming.get(0));
    }

    public ClubStats getClubStats(Long clubId) {
        BookClub club = clubRepository.findById(clubId).orElseThrow();
        int totalBooks = bookRepository.countByClubId(clubId);
        int completedBooks = bookRepository.countByClubIdAndStatus(clubId, "COMPLETED");
        List<Discussion> recentDiscussions = discussionRepository
                .findByClubIdOrderByCreatedAtDesc(clubId, 5);
        return new ClubStats(club.getName(), club.getCurrentMembers(), totalBooks,
            completedBooks, recentDiscussions.size());
    }

    public record ClubStats(String clubName, int memberCount, int totalBooks,
                             int completedBooks, int recentDiscussions) {}
}
```

### Controller

```java
// BookClubController.java
@RestController
@RequestMapping("/api/v1/book-clubs")
@RequiredArgsConstructor
public class BookClubController {

    private final BookClubService service;

    @PostMapping
    public ResponseEntity<BookClub> createClub(@RequestBody CreateClubRequest request) {
        return ResponseEntity.ok(service.createClub(request.ownerId(), request.name(),
            request.description(), request.genreFocus(), request.isPrivate()));
    }

    @PostMapping("/{clubId}/join")
    public ResponseEntity<ClubMember> joinClub(@PathVariable Long clubId,
            @RequestParam Long userId) {
        return ResponseEntity.ok(service.joinClub(clubId, userId));
    }

    @PostMapping("/{clubId}/books")
    public ResponseEntity<ClubBook> addBook(@PathVariable Long clubId,
            @RequestBody AddBookRequest request) {
        return ResponseEntity.ok(service.addBook(clubId, request.addedBy(), request.title(),
            request.author(), request.isbn(), request.totalPages(), request.isMonthlyPick(),
            request.startDate(), request.endDate()));
    }

    @PutMapping("/books/{clubBookId}/progress")
    public ResponseEntity<ReadingProgress> updateProgress(@PathVariable Long clubBookId,
            @RequestParam Long userId, @RequestParam int pagesRead) {
        return ResponseEntity.ok(service.updateProgress(userId, clubBookId, pagesRead));
    }

    @PostMapping("/{clubId}/discussions")
    public ResponseEntity<Discussion> postDiscussion(@PathVariable Long clubId,
            @RequestBody PostDiscussionRequest request) {
        return ResponseEntity.ok(service.postDiscussion(clubId, request.userId(),
            request.clubBookId(), request.title(), request.content(),
            request.spoilerWarning(), request.parentId()));
    }

    @PostMapping("/books/{clubBookId}/rate")
    public ResponseEntity<BookRating> rate(@PathVariable Long clubBookId,
            @RequestBody RateBookRequest request) {
        return ResponseEntity.ok(service.rateBook(request.userId(), clubBookId,
            request.rating(), request.review()));
    }

    @GetMapping("/{clubId}/stats")
    public ResponseEntity<BookClubService.ClubStats> getStats(@PathVariable Long clubId) {
        return ResponseEntity.ok(service.getClubStats(clubId));
    }

    record CreateClubRequest(Long ownerId, String name, String description,
                              String genreFocus, boolean isPrivate) {}
    record AddBookRequest(Long addedBy, String title, String author, String isbn,
        int totalPages, boolean isMonthlyPick, LocalDate startDate, LocalDate endDate) {}
    record PostDiscussionRequest(Long userId, Long clubBookId, String title,
        String content, boolean spoilerWarning, Long parentId) {}
    record RateBookRequest(Long userId, int rating, String review) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Book Club Platform)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: bookclub_db
      POSTGRES_USER: bookclub_user
      POSTGRES_PASSWORD: bookclub_pass
    ports:
      - "5432:5432"
    volumes:
      - bookclub_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  bookclub-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/bookclub_db
      SPRING_DATASOURCE_USERNAME: bookclub_user
      SPRING_DATASOURCE_PASSWORD: bookclub_pass
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  bookclub_pg_data:
```

---

*[← Part 113: Fitness & Wellness](./part-113-fitness-wellness.md) | [Part 115: Analytics & Monitoring →](./part-115-analytics-monitoring.md)*

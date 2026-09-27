# Part 113: โปรเจค 61-65 — Fitness & Wellness

**ระดับ:** ระดับโลก (World-Class)
**เวลา:** 10-15 ชั่วโมง
**เป้าหมาย:** สร้างแอปพลิเคชันด้านสุขภาพและความเป็นอยู่ที่ดี ครอบคลุม Fitness Tracking, Gym Booking, Telemedicine, Mental Health Journaling และ Donation Platform

---

*[← Part 112: Food & Health](./part-112-food-health.md) | [Part 114: Education & Learning →](./part-114-education-learning.md)*

---

## โปรเจค 61: Fitness Tracking App

### ภาพรวมระบบ

Fitness Tracking App ช่วยให้ผู้ใช้บันทึกการออกกำลังกาย ติดตามความก้าวหน้า และพัฒนาสมรรถภาพร่างกาย ระบบมีคลังท่าออกกำลังกาย สร้าง Routine ส่วนตัว บันทึก Personal Bests และวัดผลร่างกาย

- **Workout Logs**: บันทึก Workout แต่ละครั้งพร้อม Sets/Reps
- **Exercise Library**: คลังท่าออกกำลังกายพร้อมวิดีโอ
- **Custom Routines**: สร้าง Workout Routine ส่วนตัว
- **Progress Tracking**: ติดตามน้ำหนัก/รอบ/เวลา
- **Personal Bests**: บันทึกและแจ้งเตือนเมื่อทำ PB
- **Body Measurements**: วัดสัดส่วนร่างกาย

### Flyway Migration

```sql
-- V1__create_fitness_tables.sql
CREATE TABLE exercises (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    muscle_group VARCHAR(100),
    secondary_muscles TEXT,
    equipment VARCHAR(100),
    difficulty VARCHAR(20) DEFAULT 'BEGINNER',
    exercise_type VARCHAR(50) DEFAULT 'STRENGTH',
    video_url VARCHAR(500),
    image_url VARCHAR(500),
    instructions TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE workout_routines (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    goal VARCHAR(50),
    days_per_week INT DEFAULT 3,
    is_public BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE routine_exercises (
    id BIGSERIAL PRIMARY KEY,
    routine_id BIGINT NOT NULL REFERENCES workout_routines(id) ON DELETE CASCADE,
    exercise_id BIGINT NOT NULL REFERENCES exercises(id),
    day_number INT NOT NULL,
    set_count INT NOT NULL DEFAULT 3,
    rep_range_min INT,
    rep_range_max INT,
    rest_seconds INT DEFAULT 60,
    display_order INT DEFAULT 0
);

CREATE TABLE workout_sessions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    routine_id BIGINT REFERENCES workout_routines(id),
    name VARCHAR(200),
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    duration_minutes INT,
    notes TEXT,
    overall_feeling INT CHECK (overall_feeling BETWEEN 1 AND 5),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE workout_sets (
    id BIGSERIAL PRIMARY KEY,
    session_id BIGINT NOT NULL REFERENCES workout_sessions(id) ON DELETE CASCADE,
    exercise_id BIGINT NOT NULL REFERENCES exercises(id),
    set_number INT NOT NULL,
    weight_kg DECIMAL(8,2),
    reps INT,
    duration_seconds INT,
    distance_meters DECIMAL(10,2),
    rpe DECIMAL(3,1),
    is_pr BOOLEAN DEFAULT FALSE,
    logged_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE personal_records (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    exercise_id BIGINT NOT NULL REFERENCES exercises(id),
    record_type VARCHAR(20) NOT NULL,
    value DECIMAL(10,2) NOT NULL,
    unit VARCHAR(20),
    session_id BIGINT REFERENCES workout_sessions(id),
    achieved_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, exercise_id, record_type)
);

CREATE TABLE body_measurements (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    weight_kg DECIMAL(5,2),
    body_fat_percentage DECIMAL(5,2),
    chest_cm DECIMAL(6,2),
    waist_cm DECIMAL(6,2),
    hips_cm DECIMAL(6,2),
    biceps_cm DECIMAL(6,2),
    thighs_cm DECIMAL(6,2),
    measured_at DATE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_workout_sets_session ON workout_sets(session_id);
CREATE INDEX idx_personal_records_user_exercise ON personal_records(user_id, exercise_id);
CREATE INDEX idx_body_measurements_user ON body_measurements(user_id, measured_at DESC);
```

### Entity

```java
// Exercise.java
package com.fitness.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "exercises")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Exercise {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "muscle_group")
    private String muscleGroup;

    @Column(name = "equipment")
    private String equipment;

    @Column(name = "difficulty")
    private String difficulty = "BEGINNER";

    @Column(name = "exercise_type")
    private String exerciseType = "STRENGTH";

    @Column(name = "video_url")
    private String videoUrl;

    @Column(name = "instructions", columnDefinition = "TEXT")
    private String instructions;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

// WorkoutSession.java
@Entity
@Table(name = "workout_sessions")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class WorkoutSession {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "routine_id")
    private Long routineId;

    @Column(name = "name")
    private String name;

    @Column(name = "started_at", nullable = false)
    private java.time.LocalDateTime startedAt;

    @Column(name = "completed_at")
    private java.time.LocalDateTime completedAt;

    @Column(name = "duration_minutes")
    private Integer durationMinutes;

    @Column(name = "notes")
    private String notes;

    @Column(name = "overall_feeling")
    private Integer overallFeeling;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "session_id")
    private java.util.List<WorkoutSet> sets;

    @Column(name = "created_at")
    private java.time.LocalDateTime createdAt = java.time.LocalDateTime.now();
}

// WorkoutSet.java
@Entity
@Table(name = "workout_sets")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class WorkoutSet {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "session_id", nullable = false)
    private Long sessionId;

    @Column(name = "exercise_id", nullable = false)
    private Long exerciseId;

    @Column(name = "set_number", nullable = false)
    private Integer setNumber;

    @Column(name = "weight_kg")
    private java.math.BigDecimal weightKg;

    @Column(name = "reps")
    private Integer reps;

    @Column(name = "duration_seconds")
    private Integer durationSeconds;

    @Column(name = "is_pr")
    private Boolean isPr = false;

    @Column(name = "logged_at")
    private java.time.LocalDateTime loggedAt = java.time.LocalDateTime.now();
}

// PersonalRecord.java
@Entity
@Table(name = "personal_records")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class PersonalRecord {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "exercise_id", nullable = false)
    private Long exerciseId;

    @Column(name = "record_type", nullable = false)
    private String recordType; // MAX_WEIGHT, MAX_REPS, MAX_DURATION, MAX_DISTANCE

    @Column(name = "value", nullable = false)
    private java.math.BigDecimal value;

    @Column(name = "unit")
    private String unit;

    @Column(name = "session_id")
    private Long sessionId;

    @Column(name = "achieved_at")
    private java.time.LocalDateTime achievedAt = java.time.LocalDateTime.now();
}
```

### Service

```java
// FitnessService.java
package com.fitness.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class FitnessService {

    private final WorkoutSessionRepository sessionRepository;
    private final WorkoutSetRepository setRepository;
    private final PersonalRecordRepository prRepository;
    private final ExerciseRepository exerciseRepository;
    private final WorkoutRoutineRepository routineRepository;
    private final BodyMeasurementRepository measurementRepository;

    @Transactional
    public WorkoutSession startWorkout(Long userId, Long routineId, String name) {
        WorkoutSession session = WorkoutSession.builder()
                .userId(userId).routineId(routineId).name(name)
                .startedAt(LocalDateTime.now()).build();
        return sessionRepository.save(session);
    }

    @Transactional
    public WorkoutSet logSet(Long sessionId, Long exerciseId, int setNumber,
                              BigDecimal weightKg, Integer reps, Integer durationSeconds) {
        WorkoutSession session = sessionRepository.findById(sessionId)
                .orElseThrow(() -> new RuntimeException("Session not found"));

        WorkoutSet set = WorkoutSet.builder()
                .sessionId(sessionId).exerciseId(exerciseId)
                .setNumber(setNumber).weightKg(weightKg)
                .reps(reps).durationSeconds(durationSeconds).build();

        // Check for Personal Record
        boolean isPR = checkAndUpdatePR(session.getUserId(), exerciseId, weightKg, reps,
                                         durationSeconds, sessionId);
        set.setIsPr(isPR);
        return setRepository.save(set);
    }

    @Transactional
    public WorkoutSession completeWorkout(Long sessionId, String notes, Integer feeling) {
        WorkoutSession session = sessionRepository.findById(sessionId)
                .orElseThrow(() -> new RuntimeException("Session not found"));
        LocalDateTime completed = LocalDateTime.now();
        session.setCompletedAt(completed);
        session.setNotes(notes);
        session.setOverallFeeling(feeling);
        long minutes = java.time.Duration.between(session.getStartedAt(), completed).toMinutes();
        session.setDurationMinutes((int) minutes);
        return sessionRepository.save(session);
    }

    @Transactional
    public BodyMeasurement logMeasurement(Long userId, BigDecimal weightKg, BigDecimal bodyFat,
                                           BigDecimal chestCm, BigDecimal waistCm, BigDecimal hipsCm) {
        BodyMeasurement m = BodyMeasurement.builder()
                .userId(userId).weightKg(weightKg).bodyFatPercentage(bodyFat)
                .chestCm(chestCm).waistCm(waistCm).hipsCm(hipsCm)
                .measuredAt(java.time.LocalDate.now()).build();
        return measurementRepository.save(m);
    }

    public List<PersonalRecord> getPersonalRecords(Long userId) {
        return prRepository.findByUserId(userId);
    }

    public WorkoutStats getWorkoutStats(Long userId, int days) {
        LocalDateTime since = LocalDateTime.now().minusDays(days);
        List<WorkoutSession> sessions = sessionRepository
                .findByUserIdAndCreatedAtAfterAndCompletedAtIsNotNull(userId, since);
        int totalWorkouts = sessions.size();
        long totalMinutes = sessions.stream()
                .filter(s -> s.getDurationMinutes() != null)
                .mapToLong(WorkoutSession::getDurationMinutes).sum();
        return new WorkoutStats(totalWorkouts, totalMinutes, days);
    }

    private boolean checkAndUpdatePR(Long userId, Long exerciseId, BigDecimal weight,
                                      Integer reps, Integer duration, Long sessionId) {
        if (weight != null && reps != null) {
            Optional<PersonalRecord> existing = prRepository.findByUserIdAndExerciseIdAndRecordType(
                userId, exerciseId, "MAX_WEIGHT");
            if (existing.isEmpty() || weight.compareTo(existing.get().getValue()) > 0) {
                PersonalRecord pr = existing.orElse(PersonalRecord.builder()
                    .userId(userId).exerciseId(exerciseId).recordType("MAX_WEIGHT")
                    .unit("kg").build());
                pr.setValue(weight);
                pr.setSessionId(sessionId);
                pr.setAchievedAt(LocalDateTime.now());
                prRepository.save(pr);
                return true;
            }
        }
        return false;
    }

    public record WorkoutStats(int totalWorkouts, long totalMinutes, int periodDays) {}
}
```

### Controller

```java
// FitnessController.java
@RestController
@RequestMapping("/api/v1/fitness")
@RequiredArgsConstructor
public class FitnessController {

    private final FitnessService fitnessService;
    private final ExerciseRepository exerciseRepository;

    @GetMapping("/exercises")
    public ResponseEntity<List<Exercise>> getExercises(
            @RequestParam(required = false) String muscleGroup,
            @RequestParam(required = false) String equipment) {
        return ResponseEntity.ok(exerciseRepository.findByFilters(muscleGroup, equipment));
    }

    @PostMapping("/sessions")
    public ResponseEntity<WorkoutSession> startWorkout(@RequestBody StartWorkoutRequest request) {
        return ResponseEntity.ok(fitnessService.startWorkout(
            request.userId(), request.routineId(), request.name()));
    }

    @PostMapping("/sessions/{sessionId}/sets")
    public ResponseEntity<WorkoutSet> logSet(@PathVariable Long sessionId,
            @RequestBody LogSetRequest request) {
        return ResponseEntity.ok(fitnessService.logSet(sessionId, request.exerciseId(),
            request.setNumber(), request.weightKg(), request.reps(), request.durationSeconds()));
    }

    @PutMapping("/sessions/{sessionId}/complete")
    public ResponseEntity<WorkoutSession> completeWorkout(@PathVariable Long sessionId,
            @RequestBody CompleteWorkoutRequest request) {
        return ResponseEntity.ok(fitnessService.completeWorkout(
            sessionId, request.notes(), request.feeling()));
    }

    @PostMapping("/measurements")
    public ResponseEntity<BodyMeasurement> logMeasurement(@RequestBody LogMeasurementRequest req) {
        return ResponseEntity.ok(fitnessService.logMeasurement(req.userId(), req.weightKg(),
            req.bodyFat(), req.chestCm(), req.waistCm(), req.hipsCm()));
    }

    @GetMapping("/stats/{userId}")
    public ResponseEntity<FitnessService.WorkoutStats> getStats(@PathVariable Long userId,
            @RequestParam(defaultValue = "30") int days) {
        return ResponseEntity.ok(fitnessService.getWorkoutStats(userId, days));
    }

    record StartWorkoutRequest(Long userId, Long routineId, String name) {}
    record LogSetRequest(Long exerciseId, int setNumber, java.math.BigDecimal weightKg,
                          Integer reps, Integer durationSeconds) {}
    record CompleteWorkoutRequest(String notes, Integer feeling) {}
    record LogMeasurementRequest(Long userId, java.math.BigDecimal weightKg,
        java.math.BigDecimal bodyFat, java.math.BigDecimal chestCm,
        java.math.BigDecimal waistCm, java.math.BigDecimal hipsCm) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Fitness Tracking App)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: fitness_db
      POSTGRES_USER: fitness_user
      POSTGRES_PASSWORD: fitness_pass
    ports:
      - "5432:5432"
    volumes:
      - fitness_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  fitness-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/fitness_db
      SPRING_DATASOURCE_USERNAME: fitness_user
      SPRING_DATASOURCE_PASSWORD: fitness_pass
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  fitness_pg_data:
```

---

## โปรเจค 62: Gym Class Booking System

### ภาพรวมระบบ

ระบบจองคลาส Gym ที่ครบครัน จัดการตารางเรียน ผู้สอน ความจุ Waitlist และประวัติการเข้าเรียน เหมาะสำหรับ Fitness Studio, Yoga Studio หรือ Gym ขนาดใหญ่

- **Class Schedule**: จัดการตารางคลาสและผู้สอน
- **Capacity Management**: ควบคุมจำนวนที่นั่งต่อคลาส
- **Waitlist**: รายชื่อรอเมื่อคลาสเต็ม
- **Booking/Cancellation**: จองและยกเลิกพร้อม Notification
- **Attendance**: บันทึกการเข้าเรียน

### Flyway Migration

```sql
-- V1__create_gym_booking_tables.sql
CREATE TABLE instructors (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE,
    name VARCHAR(200) NOT NULL,
    bio TEXT,
    specialties TEXT,
    image_url VARCHAR(500),
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_ratings INT DEFAULT 0,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE class_types (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    duration_minutes INT NOT NULL DEFAULT 60,
    intensity_level VARCHAR(20) DEFAULT 'MEDIUM',
    calories_burned_estimate INT,
    image_url VARCHAR(500),
    active BOOLEAN DEFAULT TRUE
);

CREATE TABLE class_schedules (
    id BIGSERIAL PRIMARY KEY,
    class_type_id BIGINT NOT NULL REFERENCES class_types(id),
    instructor_id BIGINT NOT NULL REFERENCES instructors(id),
    room VARCHAR(100),
    starts_at TIMESTAMP NOT NULL,
    ends_at TIMESTAMP NOT NULL,
    capacity INT NOT NULL DEFAULT 20,
    enrolled_count INT NOT NULL DEFAULT 0,
    waitlist_count INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED',
    notes TEXT,
    is_recurring BOOLEAN DEFAULT FALSE,
    recurrence_rule VARCHAR(200),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE bookings (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    schedule_id BIGINT NOT NULL REFERENCES class_schedules(id),
    status VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED',
    is_waitlisted BOOLEAN NOT NULL DEFAULT FALSE,
    waitlist_position INT,
    booked_at TIMESTAMP NOT NULL DEFAULT NOW(),
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(500),
    attended BOOLEAN,
    check_in_at TIMESTAMP
);

CREATE TABLE class_series (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    total_sessions INT NOT NULL,
    valid_for_days INT NOT NULL DEFAULT 30,
    price DECIMAL(10,2) NOT NULL,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE user_series_enrollments (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    series_id BIGINT NOT NULL REFERENCES class_series(id),
    sessions_remaining INT NOT NULL,
    purchased_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    UNIQUE (user_id, series_id)
);

CREATE UNIQUE INDEX idx_bookings_user_schedule ON bookings(user_id, schedule_id)
    WHERE status != 'CANCELLED';
CREATE INDEX idx_schedules_starts_at ON class_schedules(starts_at);
CREATE INDEX idx_bookings_user ON bookings(user_id);
```

### Entity & Service

```java
// ClassSchedule.java
package com.gymbooking.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "class_schedules")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class ClassSchedule {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "class_type_id", nullable = false)
    private Long classTypeId;

    @Column(name = "instructor_id", nullable = false)
    private Long instructorId;

    @Column(name = "room")
    private String room;

    @Column(name = "starts_at", nullable = false)
    private LocalDateTime startsAt;

    @Column(name = "ends_at", nullable = false)
    private LocalDateTime endsAt;

    @Column(name = "capacity", nullable = false)
    private Integer capacity = 20;

    @Column(name = "enrolled_count", nullable = false)
    private Integer enrolledCount = 0;

    @Column(name = "waitlist_count", nullable = false)
    private Integer waitlistCount = 0;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private ScheduleStatus status = ScheduleStatus.SCHEDULED;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}

public enum ScheduleStatus { SCHEDULED, CANCELLED, COMPLETED, IN_PROGRESS }

// Booking.java
@Entity
@Table(name = "bookings")
@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Booking {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "schedule_id", nullable = false)
    private Long scheduleId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private BookingStatus status = BookingStatus.CONFIRMED;

    @Column(name = "is_waitlisted", nullable = false)
    private Boolean isWaitlisted = false;

    @Column(name = "waitlist_position")
    private Integer waitlistPosition;

    @Column(name = "booked_at")
    private LocalDateTime bookedAt = LocalDateTime.now();

    @Column(name = "cancelled_at")
    private LocalDateTime cancelledAt;

    @Column(name = "attended")
    private Boolean attended;

    @Column(name = "check_in_at")
    private LocalDateTime checkInAt;
}

public enum BookingStatus { CONFIRMED, WAITLISTED, CANCELLED, ATTENDED, NO_SHOW }

// GymBookingService.java
@Service
@RequiredArgsConstructor
public class GymBookingService {

    private final ClassScheduleRepository scheduleRepository;
    private final BookingRepository bookingRepository;
    private final NotificationService notificationService;

    @Transactional
    public Booking bookClass(Long userId, Long scheduleId) {
        ClassSchedule schedule = scheduleRepository.findByIdWithLock(scheduleId)
                .orElseThrow(() -> new RuntimeException("Class not found"));

        if (schedule.getStatus() != ScheduleStatus.SCHEDULED) {
            throw new IllegalStateException("Class is not available for booking");
        }
        if (schedule.getStartsAt().isBefore(LocalDateTime.now())) {
            throw new IllegalStateException("Cannot book a past class");
        }
        if (bookingRepository.existsByUserIdAndScheduleIdAndStatusNot(
                userId, scheduleId, BookingStatus.CANCELLED)) {
            throw new IllegalStateException("Already booked this class");
        }

        boolean isWaitlisted = schedule.getEnrolledCount() >= schedule.getCapacity();
        Booking booking = Booking.builder()
                .userId(userId).scheduleId(scheduleId)
                .status(isWaitlisted ? BookingStatus.WAITLISTED : BookingStatus.CONFIRMED)
                .isWaitlisted(isWaitlisted).build();

        if (isWaitlisted) {
            schedule.setWaitlistCount(schedule.getWaitlistCount() + 1);
            booking.setWaitlistPosition(schedule.getWaitlistCount());
        } else {
            schedule.setEnrolledCount(schedule.getEnrolledCount() + 1);
        }
        scheduleRepository.save(schedule);
        Booking saved = bookingRepository.save(booking);
        notificationService.sendBookingConfirmation(userId, schedule, isWaitlisted);
        return saved;
    }

    @Transactional
    public void cancelBooking(Long bookingId, Long userId, String reason) {
        Booking booking = bookingRepository.findById(bookingId)
                .orElseThrow(() -> new RuntimeException("Booking not found"));
        if (!booking.getUserId().equals(userId)) {
            throw new SecurityException("Unauthorized to cancel this booking");
        }
        if (booking.getStatus() == BookingStatus.CANCELLED) {
            throw new IllegalStateException("Booking already cancelled");
        }

        ClassSchedule schedule = scheduleRepository.findById(booking.getScheduleId()).orElseThrow();
        boolean wasWaitlisted = booking.getIsWaitlisted();

        booking.setStatus(BookingStatus.CANCELLED);
        booking.setCancelledAt(LocalDateTime.now());
        bookingRepository.save(booking);

        if (!wasWaitlisted) {
            schedule.setEnrolledCount(Math.max(0, schedule.getEnrolledCount() - 1));
            scheduleRepository.save(schedule);
            promoteFromWaitlist(schedule);
        } else {
            schedule.setWaitlistCount(Math.max(0, schedule.getWaitlistCount() - 1));
            scheduleRepository.save(schedule);
        }
    }

    @Transactional
    public void checkIn(Long bookingId) {
        Booking booking = bookingRepository.findById(bookingId)
                .orElseThrow(() -> new RuntimeException("Booking not found"));
        if (booking.getStatus() != BookingStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot check in: status is " + booking.getStatus());
        }
        booking.setStatus(BookingStatus.ATTENDED);
        booking.setAttended(true);
        booking.setCheckInAt(LocalDateTime.now());
        bookingRepository.save(booking);
    }

    private void promoteFromWaitlist(ClassSchedule schedule) {
        List<Booking> waitlisted = bookingRepository
                .findByScheduleIdAndStatusOrderByBookedAt(schedule.getId(), BookingStatus.WAITLISTED);
        if (!waitlisted.isEmpty() && schedule.getEnrolledCount() < schedule.getCapacity()) {
            Booking promoted = waitlisted.get(0);
            promoted.setStatus(BookingStatus.CONFIRMED);
            promoted.setIsWaitlisted(false);
            promoted.setWaitlistPosition(null);
            bookingRepository.save(promoted);
            schedule.setEnrolledCount(schedule.getEnrolledCount() + 1);
            schedule.setWaitlistCount(Math.max(0, schedule.getWaitlistCount() - 1));
            scheduleRepository.save(schedule);
            notificationService.sendWaitlistPromotionNotification(promoted.getUserId(), schedule);
        }
    }

    public List<ClassSchedule> getUpcomingClasses(java.time.LocalDate date) {
        LocalDateTime start = date.atStartOfDay();
        LocalDateTime end = date.atTime(23, 59, 59);
        return scheduleRepository.findByStartsAtBetweenAndStatusOrderByStartsAt(
            start, end, ScheduleStatus.SCHEDULED);
    }
}
```

### Controller

```java
// GymBookingController.java
@RestController
@RequestMapping("/api/v1/gym")
@RequiredArgsConstructor
public class GymBookingController {

    private final GymBookingService bookingService;

    @GetMapping("/classes")
    public ResponseEntity<List<ClassSchedule>> getClasses(
            @RequestParam(required = false) java.time.LocalDate date) {
        return ResponseEntity.ok(bookingService.getUpcomingClasses(
            date != null ? date : java.time.LocalDate.now()));
    }

    @PostMapping("/bookings")
    public ResponseEntity<Booking> bookClass(@RequestBody BookClassRequest request) {
        return ResponseEntity.ok(bookingService.bookClass(request.userId(), request.scheduleId()));
    }

    @PutMapping("/bookings/{bookingId}/cancel")
    public ResponseEntity<Void> cancelBooking(@PathVariable Long bookingId,
            @RequestParam Long userId, @RequestParam(required = false) String reason) {
        bookingService.cancelBooking(bookingId, userId, reason);
        return ResponseEntity.ok().build();
    }

    @PostMapping("/bookings/{bookingId}/checkin")
    public ResponseEntity<Void> checkIn(@PathVariable Long bookingId) {
        bookingService.checkIn(bookingId);
        return ResponseEntity.ok().build();
    }

    record BookClassRequest(Long userId, Long scheduleId) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Gym Class Booking)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: gym_db
      POSTGRES_USER: gym_user
      POSTGRES_PASSWORD: gym_pass
    ports:
      - "5432:5432"
    volumes:
      - gym_pg_data:/var/lib/postgresql/data

  gym-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/gym_db
      SPRING_DATASOURCE_USERNAME: gym_user
      SPRING_DATASOURCE_PASSWORD: gym_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  gym_pg_data:
```

---

## โปรเจค 63: Telemedicine Platform

### ภาพรวมระบบ

Telemedicine Platform เชื่อมต่อแพทย์และผู้ป่วยผ่านระบบนัดหมายออนไลน์ รองรับการปรึกษาทางวิดีโอ บันทึกประวัติการรักษา การออกใบสั่งยา และการนัดติดตาม

- **Doctor Profiles**: โปรไฟล์แพทย์พร้อมความเชี่ยวชาญ
- **Appointment Booking**: ระบบนัดหมายและจัดการตารางแพทย์
- **Medical Records**: บันทึกประวัติการรักษา
- **Prescriptions**: ออกใบสั่งยาดิจิทัล
- **Follow-up Scheduling**: นัดติดตามอาการ

### Flyway Migration

```sql
-- V1__create_telemedicine_tables.sql
CREATE TABLE doctors (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    title VARCHAR(50),
    specialization VARCHAR(200),
    license_number VARCHAR(100) NOT NULL UNIQUE,
    years_experience INT,
    bio TEXT,
    languages TEXT,
    consultation_fee DECIMAL(10,2),
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_consultations INT DEFAULT 0,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE doctor_availability (
    id BIGSERIAL PRIMARY KEY,
    doctor_id BIGINT NOT NULL REFERENCES doctors(id),
    day_of_week INT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    slot_duration_minutes INT NOT NULL DEFAULT 30,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    appointment_number VARCHAR(30) NOT NULL UNIQUE,
    patient_id BIGINT NOT NULL,
    doctor_id BIGINT NOT NULL REFERENCES doctors(id),
    appointment_type VARCHAR(30) NOT NULL DEFAULT 'VIDEO',
    status VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED',
    scheduled_at TIMESTAMP NOT NULL,
    duration_minutes INT NOT NULL DEFAULT 30,
    chief_complaint TEXT,
    video_room_id VARCHAR(200),
    notes TEXT,
    actual_started_at TIMESTAMP,
    actual_ended_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE medical_records (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL,
    appointment_id BIGINT REFERENCES appointments(id),
    doctor_id BIGINT NOT NULL REFERENCES doctors(id),
    diagnosis TEXT,
    symptoms TEXT,
    examination_notes TEXT,
    recommendations TEXT,
    follow_up_date DATE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE prescriptions (
    id BIGSERIAL PRIMARY KEY,
    record_id BIGINT NOT NULL REFERENCES medical_records(id),
    patient_id BIGINT NOT NULL,
    doctor_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    valid_until DATE NOT NULL,
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE prescription_medications (
    id BIGSERIAL PRIMARY KEY,
    prescription_id BIGINT NOT NULL REFERENCES prescriptions(id),
    medication_name VARCHAR(300) NOT NULL,
    dosage VARCHAR(200),
    frequency VARCHAR(200),
    duration VARCHAR(100),
    instructions TEXT
);

CREATE INDEX idx_appointments_doctor_status ON appointments(doctor_id, status);
CREATE INDEX idx_appointments_patient ON appointments(patient_id);
CREATE INDEX idx_medical_records_patient ON medical_records(patient_id);
```

### Entity & Service

```java
// TelemedicineService.java
package com.telemedicine.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class TelemedicineService {

    private final DoctorRepository doctorRepository;
    private final AppointmentRepository appointmentRepository;
    private final MedicalRecordRepository recordRepository;
    private final PrescriptionRepository prescriptionRepository;
    private final VideoService videoService;
    private final NotificationService notificationService;

    @Transactional
    public Appointment bookAppointment(Long patientId, Long doctorId,
                                        LocalDateTime scheduledAt, String chiefComplaint,
                                        String appointmentType) {
        Doctor doctor = doctorRepository.findById(doctorId)
                .orElseThrow(() -> new RuntimeException("Doctor not found"));
        if (!doctor.getActive()) {
            throw new IllegalStateException("Doctor is not available");
        }
        // Check for conflicting appointment
        boolean conflict = appointmentRepository.existsByDoctorIdAndScheduledAtAndStatusNot(
            doctorId, scheduledAt, "CANCELLED");
        if (conflict) {
            throw new IllegalStateException("Time slot is already booked");
        }

        String appointmentNumber = "APT-" + System.currentTimeMillis();
        String videoRoomId = UUID.randomUUID().toString();

        Appointment appointment = Appointment.builder()
                .appointmentNumber(appointmentNumber)
                .patientId(patientId).doctorId(doctorId)
                .appointmentType(appointmentType != null ? appointmentType : "VIDEO")
                .status("SCHEDULED")
                .scheduledAt(scheduledAt)
                .durationMinutes(doctor.getConsultationDurationMinutes() != null ?
                    doctor.getConsultationDurationMinutes() : 30)
                .chiefComplaint(chiefComplaint)
                .videoRoomId(videoRoomId)
                .build();
        appointment = appointmentRepository.save(appointment);
        notificationService.sendAppointmentConfirmation(patientId, doctorId, appointment);
        return appointment;
    }

    @Transactional
    public MedicalRecord createMedicalRecord(Long appointmentId, Long doctorId,
                                              String diagnosis, String symptoms,
                                              String recommendations, LocalDate followUpDate) {
        Appointment appointment = appointmentRepository.findById(appointmentId)
                .orElseThrow(() -> new RuntimeException("Appointment not found"));

        MedicalRecord record = MedicalRecord.builder()
                .patientId(appointment.getPatientId())
                .appointmentId(appointmentId)
                .doctorId(doctorId)
                .diagnosis(diagnosis)
                .symptoms(symptoms)
                .recommendations(recommendations)
                .followUpDate(followUpDate)
                .build();
        record = recordRepository.save(record);

        appointment.setStatus("COMPLETED");
        appointment.setActualEndedAt(LocalDateTime.now());
        appointmentRepository.save(appointment);

        return record;
    }

    @Transactional
    public Prescription issuePrescription(Long recordId, Long patientId, Long doctorId,
                                           LocalDate validUntil,
                                           List<MedicationRequest> medications) {
        Prescription prescription = Prescription.builder()
                .recordId(recordId).patientId(patientId).doctorId(doctorId)
                .status("ACTIVE").validUntil(validUntil).build();
        prescription = prescriptionRepository.save(prescription);

        for (MedicationRequest med : medications) {
            PrescriptionMedication medication = PrescriptionMedication.builder()
                    .prescriptionId(prescription.getId())
                    .medicationName(med.name()).dosage(med.dosage())
                    .frequency(med.frequency()).duration(med.duration())
                    .instructions(med.instructions()).build();
            prescriptionRepository.saveMedication(medication);
        }
        return prescription;
    }

    public List<Doctor> searchDoctors(String specialization, String language) {
        if (specialization != null && language != null) {
            return doctorRepository.findBySpecializationContainingAndLanguagesContainingAndActiveTrue(
                specialization, language);
        }
        if (specialization != null) {
            return doctorRepository.findBySpecializationContainingAndActiveTrue(specialization);
        }
        return doctorRepository.findByActiveTrueOrderByRatingDesc();
    }

    public String getVideoRoomToken(Long appointmentId, Long userId) {
        Appointment appointment = appointmentRepository.findById(appointmentId)
                .orElseThrow(() -> new RuntimeException("Appointment not found"));
        if (!appointment.getPatientId().equals(userId) &&
            !appointment.getDoctorId().equals(userId)) {
            throw new SecurityException("Not authorized for this appointment");
        }
        return videoService.generateToken(appointment.getVideoRoomId(), userId);
    }

    public record MedicationRequest(String name, String dosage, String frequency,
                                     String duration, String instructions) {}
}
```

### Controller

```java
// TelemedicineController.java
@RestController
@RequestMapping("/api/v1/telemedicine")
@RequiredArgsConstructor
public class TelemedicineController {

    private final TelemedicineService service;

    @GetMapping("/doctors")
    public ResponseEntity<List<Doctor>> searchDoctors(
            @RequestParam(required = false) String specialization,
            @RequestParam(required = false) String language) {
        return ResponseEntity.ok(service.searchDoctors(specialization, language));
    }

    @PostMapping("/appointments")
    public ResponseEntity<Appointment> book(@RequestBody BookAppointmentRequest request) {
        return ResponseEntity.ok(service.bookAppointment(request.patientId(), request.doctorId(),
            request.scheduledAt(), request.chiefComplaint(), request.appointmentType()));
    }

    @PostMapping("/appointments/{appointmentId}/records")
    public ResponseEntity<MedicalRecord> createRecord(@PathVariable Long appointmentId,
            @RequestBody CreateRecordRequest request) {
        return ResponseEntity.ok(service.createMedicalRecord(appointmentId, request.doctorId(),
            request.diagnosis(), request.symptoms(), request.recommendations(), request.followUpDate()));
    }

    @PostMapping("/records/{recordId}/prescriptions")
    public ResponseEntity<Prescription> issuePrescription(@PathVariable Long recordId,
            @RequestBody IssuePrescriptionRequest request) {
        return ResponseEntity.ok(service.issuePrescription(recordId, request.patientId(),
            request.doctorId(), request.validUntil(), request.medications()));
    }

    @GetMapping("/appointments/{appointmentId}/video-token")
    public ResponseEntity<Map<String, String>> getVideoToken(@PathVariable Long appointmentId,
            @RequestParam Long userId) {
        String token = service.getVideoRoomToken(appointmentId, userId);
        return ResponseEntity.ok(Map.of("token", token));
    }

    record BookAppointmentRequest(Long patientId, Long doctorId,
        java.time.LocalDateTime scheduledAt, String chiefComplaint, String appointmentType) {}
    record CreateRecordRequest(Long doctorId, String diagnosis, String symptoms,
        String recommendations, java.time.LocalDate followUpDate) {}
    record IssuePrescriptionRequest(Long patientId, Long doctorId,
        java.time.LocalDate validUntil, List<TelemedicineService.MedicationRequest> medications) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Telemedicine Platform)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: telemedicine_db
      POSTGRES_USER: tele_user
      POSTGRES_PASSWORD: tele_pass
    ports:
      - "5432:5432"
    volumes:
      - tele_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  telemedicine-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/telemedicine_db
      SPRING_DATASOURCE_USERNAME: tele_user
      SPRING_DATASOURCE_PASSWORD: tele_pass
      SPRING_REDIS_HOST: redis
      VIDEO_SERVICE_API_KEY: ${VIDEO_API_KEY}
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

volumes:
  tele_pg_data:
```

---

## โปรเจค 64: Mental Health Journaling App

### ภาพรวมระบบ

Mental Health Journaling App ช่วยให้ผู้ใช้บันทึกความรู้สึก ติดตาม Mood, Habit และ Streak พร้อม Reflection Prompts ที่จะกระตุ้นให้ผู้ใช้ทบทวนตัวเอง ข้อมูลทั้งหมดถูกออกแบบมาให้มีความเป็นส่วนตัวสูง

- **Journal Entries**: บันทึกไดอารี่ส่วนตัว
- **Mood Tracking**: ติดตาม Mood ประจำวัน
- **Habit Tracking**: ติดตาม Habits ที่ต้องการสร้าง
- **Streak Tracking**: นับวันต่อเนื่อง
- **Reflection Prompts**: คำถามกระตุ้นการทบทวน

### Flyway Migration

```sql
-- V1__create_mental_health_tables.sql
CREATE TABLE journal_entries (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(300),
    content TEXT NOT NULL,
    mood_score INT CHECK (mood_score BETWEEN 1 AND 10),
    mood_label VARCHAR(50),
    tags TEXT,
    is_encrypted BOOLEAN DEFAULT FALSE,
    entry_date DATE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE mood_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    mood_score INT NOT NULL CHECK (mood_score BETWEEN 1 AND 10),
    mood_label VARCHAR(50),
    energy_level INT CHECK (energy_level BETWEEN 1 AND 5),
    anxiety_level INT CHECK (anxiety_level BETWEEN 1 AND 5),
    notes TEXT,
    activities TEXT,
    log_date DATE NOT NULL,
    logged_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE habits (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    frequency_type VARCHAR(20) NOT NULL DEFAULT 'DAILY',
    target_count INT NOT NULL DEFAULT 1,
    color VARCHAR(20),
    icon VARCHAR(50),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE habit_completions (
    id BIGSERIAL PRIMARY KEY,
    habit_id BIGINT NOT NULL REFERENCES habits(id),
    user_id BIGINT NOT NULL,
    completion_date DATE NOT NULL,
    count INT NOT NULL DEFAULT 1,
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (habit_id, completion_date)
);

CREATE TABLE habit_streaks (
    id BIGSERIAL PRIMARY KEY,
    habit_id BIGINT NOT NULL REFERENCES habits(id) UNIQUE,
    user_id BIGINT NOT NULL,
    current_streak INT NOT NULL DEFAULT 0,
    longest_streak INT NOT NULL DEFAULT 0,
    last_completed_date DATE,
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE reflection_prompts (
    id BIGSERIAL PRIMARY KEY,
    prompt_text TEXT NOT NULL,
    category VARCHAR(50),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO reflection_prompts (prompt_text, category) VALUES
('วันนี้คุณรู้สึกขอบคุณอะไรบ้าง?', 'GRATITUDE'),
('ความท้าทายที่คุณเผชิญวันนี้คืออะไร และคุณรับมือกับมันอย่างไร?', 'CHALLENGE'),
('อะไรทำให้คุณยิ้มได้วันนี้?', 'POSITIVE'),
('เป้าหมายที่คุณต้องการทำให้สำเร็จพรุ่งนี้คืออะไร?', 'GOAL'),
('คุณได้เรียนรู้อะไรใหม่จากวันนี้บ้าง?', 'LEARNING');
```

### Entity & Service

```java
// MentalHealthService.java
package com.mentalhealth.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;

@Service
@RequiredArgsConstructor
public class MentalHealthService {

    private final JournalEntryRepository journalRepository;
    private final MoodLogRepository moodLogRepository;
    private final HabitRepository habitRepository;
    private final HabitCompletionRepository completionRepository;
    private final HabitStreakRepository streakRepository;
    private final ReflectionPromptRepository promptRepository;

    @Transactional
    public JournalEntry createEntry(Long userId, String title, String content,
                                     Integer moodScore, String moodLabel, String tags) {
        JournalEntry entry = JournalEntry.builder()
                .userId(userId).title(title).content(content)
                .moodScore(moodScore).moodLabel(moodLabel)
                .tags(tags).entryDate(LocalDate.now()).build();
        return journalRepository.save(entry);
    }

    @Transactional
    public MoodLog logMood(Long userId, int score, String label, int energy, int anxiety,
                            String notes, String activities) {
        MoodLog log = MoodLog.builder()
                .userId(userId).moodScore(score).moodLabel(label)
                .energyLevel(energy).anxietyLevel(anxiety)
                .notes(notes).activities(activities)
                .logDate(LocalDate.now()).build();
        return moodLogRepository.save(log);
    }

    @Transactional
    public Habit createHabit(Long userId, String name, String description,
                               String frequencyType, int targetCount, String color) {
        Habit habit = Habit.builder()
                .userId(userId).name(name).description(description)
                .frequencyType(frequencyType).targetCount(targetCount).color(color).build();
        habit = habitRepository.save(habit);

        HabitStreak streak = HabitStreak.builder()
                .habitId(habit.getId()).userId(userId)
                .currentStreak(0).longestStreak(0).build();
        streakRepository.save(streak);
        return habit;
    }

    @Transactional
    public HabitCompletion completeHabit(Long habitId, Long userId) {
        LocalDate today = LocalDate.now();
        Optional<HabitCompletion> existing = completionRepository
                .findByHabitIdAndCompletionDate(habitId, today);

        HabitCompletion completion;
        if (existing.isPresent()) {
            completion = existing.get();
            completion.setCount(completion.getCount() + 1);
        } else {
            completion = HabitCompletion.builder()
                    .habitId(habitId).userId(userId)
                    .completionDate(today).count(1).build();
        }
        completion = completionRepository.save(completion);

        // Update streak
        updateStreak(habitId, userId, today);
        return completion;
    }

    private void updateStreak(Long habitId, Long userId, LocalDate today) {
        HabitStreak streak = streakRepository.findByHabitId(habitId)
                .orElse(HabitStreak.builder().habitId(habitId).userId(userId)
                    .currentStreak(0).longestStreak(0).build());

        LocalDate lastCompleted = streak.getLastCompletedDate();
        if (lastCompleted == null || lastCompleted.isBefore(today.minusDays(1))) {
            streak.setCurrentStreak(1);
        } else if (lastCompleted.equals(today.minusDays(1))) {
            streak.setCurrentStreak(streak.getCurrentStreak() + 1);
        }
        // Don't count same day twice
        if (lastCompleted != null && lastCompleted.equals(today)) return;

        if (streak.getCurrentStreak() > streak.getLongestStreak()) {
            streak.setLongestStreak(streak.getCurrentStreak());
        }
        streak.setLastCompletedDate(today);
        streak.setUpdatedAt(LocalDateTime.now());
        streakRepository.save(streak);
    }

    public ReflectionPrompt getRandomPrompt(String category) {
        List<ReflectionPrompt> prompts = category != null ?
            promptRepository.findByCategoryAndActiveTrue(category) :
            promptRepository.findByActiveTrue();
        if (prompts.isEmpty()) return null;
        return prompts.get(new Random().nextInt(prompts.size()));
    }

    public MoodAnalysis analyzeMood(Long userId, int days) {
        LocalDate from = LocalDate.now().minusDays(days);
        List<MoodLog> logs = moodLogRepository.findByUserIdAndLogDateAfter(userId, from);
        if (logs.isEmpty()) return new MoodAnalysis(0.0, 0.0, 0, Map.of());

        double avgMood = logs.stream().mapToInt(MoodLog::getMoodScore).average().orElse(0);
        double avgEnergy = logs.stream()
                .filter(l -> l.getEnergyLevel() != null)
                .mapToInt(MoodLog::getEnergyLevel).average().orElse(0);

        Map<String, Long> moodDistribution = new HashMap<>();
        for (MoodLog log : logs) {
            if (log.getMoodLabel() != null) {
                moodDistribution.merge(log.getMoodLabel(), 1L, Long::sum);
            }
        }
        return new MoodAnalysis(avgMood, avgEnergy, logs.size(), moodDistribution);
    }

    public record MoodAnalysis(double avgMoodScore, double avgEnergyLevel,
                                int totalLogs, Map<String, Long> moodDistribution) {}
}
```

### Controller & docker-compose.yml

```java
// MentalHealthController.java
@RestController
@RequestMapping("/api/v1/mental-health")
@RequiredArgsConstructor
public class MentalHealthController {

    private final MentalHealthService service;

    @PostMapping("/journal")
    public ResponseEntity<JournalEntry> createEntry(@RequestBody CreateEntryRequest request) {
        return ResponseEntity.ok(service.createEntry(request.userId(), request.title(),
            request.content(), request.moodScore(), request.moodLabel(), request.tags()));
    }

    @PostMapping("/mood")
    public ResponseEntity<MoodLog> logMood(@RequestBody LogMoodRequest request) {
        return ResponseEntity.ok(service.logMood(request.userId(), request.score(),
            request.label(), request.energy(), request.anxiety(), request.notes(), request.activities()));
    }

    @PostMapping("/habits")
    public ResponseEntity<Habit> createHabit(@RequestBody CreateHabitRequest request) {
        return ResponseEntity.ok(service.createHabit(request.userId(), request.name(),
            request.description(), request.frequencyType(), request.targetCount(), request.color()));
    }

    @PostMapping("/habits/{habitId}/complete")
    public ResponseEntity<HabitCompletion> complete(@PathVariable Long habitId,
            @RequestParam Long userId) {
        return ResponseEntity.ok(service.completeHabit(habitId, userId));
    }

    @GetMapping("/prompts/random")
    public ResponseEntity<ReflectionPrompt> getPrompt(
            @RequestParam(required = false) String category) {
        return ResponseEntity.ok(service.getRandomPrompt(category));
    }

    @GetMapping("/mood/analysis/{userId}")
    public ResponseEntity<MentalHealthService.MoodAnalysis> analyzeMood(
            @PathVariable Long userId, @RequestParam(defaultValue = "30") int days) {
        return ResponseEntity.ok(service.analyzeMood(userId, days));
    }

    record CreateEntryRequest(Long userId, String title, String content,
        Integer moodScore, String moodLabel, String tags) {}
    record LogMoodRequest(Long userId, int score, String label, int energy, int anxiety,
        String notes, String activities) {}
    record CreateHabitRequest(Long userId, String name, String description,
        String frequencyType, int targetCount, String color) {}
}
```

```yaml
# docker-compose.yml (Mental Health Journaling App)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mentalhealth_db
      POSTGRES_USER: mh_user
      POSTGRES_PASSWORD: mh_pass
    ports:
      - "5432:5432"
    volumes:
      - mh_pg_data:/var/lib/postgresql/data

  mentalhealth-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mentalhealth_db
      SPRING_DATASOURCE_USERNAME: mh_user
      SPRING_DATASOURCE_PASSWORD: mh_pass
      ENCRYPTION_KEY: ${JOURNAL_ENCRYPTION_KEY}
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  mh_pg_data:
```

---

## โปรเจค 65: Donation & Fundraising Platform

### ภาพรวมระบบ

Donation & Fundraising Platform ช่วยให้องค์กรและบุคคลสามารถระดมทุนออนไลน์ได้ง่าย ผู้บริจาคสามารถติดตามความก้าวหน้าของแคมเปญ บริจาคแบบ Recurring และรับใบเสร็จภาษี ระบบจัดการการเบิกจ่ายเงินให้กับผู้รับอย่างโปร่งใส

### Flyway Migration

```sql
-- V1__create_fundraising_tables.sql
CREATE TABLE campaigns (
    id BIGSERIAL PRIMARY KEY,
    organizer_id BIGINT NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT NOT NULL,
    short_description VARCHAR(500),
    image_url VARCHAR(500),
    goal_amount DECIMAL(14,2) NOT NULL,
    raised_amount DECIMAL(14,2) NOT NULL DEFAULT 0,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    category VARCHAR(100),
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    is_featured BOOLEAN DEFAULT FALSE,
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    donor_count INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE donations (
    id BIGSERIAL PRIMARY KEY,
    campaign_id BIGINT NOT NULL REFERENCES campaigns(id),
    donor_id BIGINT,
    donor_name VARCHAR(200),
    donor_email VARCHAR(255),
    amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    is_anonymous BOOLEAN DEFAULT FALSE,
    message TEXT,
    payment_status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    payment_reference VARCHAR(200),
    payment_method VARCHAR(50),
    is_recurring BOOLEAN DEFAULT FALSE,
    recurring_interval VARCHAR(20),
    tax_receipt_issued BOOLEAN DEFAULT FALSE,
    tax_receipt_number VARCHAR(100),
    donated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE recurring_donations (
    id BIGSERIAL PRIMARY KEY,
    original_donation_id BIGINT NOT NULL REFERENCES donations(id),
    campaign_id BIGINT NOT NULL REFERENCES campaigns(id),
    donor_id BIGINT,
    donor_email VARCHAR(255),
    amount DECIMAL(12,2) NOT NULL,
    interval_type VARCHAR(20) NOT NULL,
    next_charge_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    total_charges INT DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE fund_disbursements (
    id BIGSERIAL PRIMARY KEY,
    campaign_id BIGINT NOT NULL REFERENCES campaigns(id),
    amount DECIMAL(12,2) NOT NULL,
    recipient_name VARCHAR(200) NOT NULL,
    recipient_account VARCHAR(200),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    description TEXT,
    processed_by BIGINT,
    processed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_donations_campaign ON donations(campaign_id);
CREATE INDEX idx_campaigns_status ON campaigns(status);
CREATE INDEX idx_recurring_donations_next_charge ON recurring_donations(next_charge_date, status);
```

### Entity & Service

```java
// FundraisingService.java
package com.fundraising.service;

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
public class FundraisingService {

    private final CampaignRepository campaignRepository;
    private final DonationRepository donationRepository;
    private final RecurringDonationRepository recurringRepository;
    private final FundDisbursementRepository disbursementRepository;
    private final PaymentService paymentService;
    private final EmailService emailService;

    @Transactional
    public Campaign createCampaign(Long organizerId, String title, String description,
                                    BigDecimal goalAmount, String currency, String category,
                                    LocalDateTime startDate, LocalDateTime endDate) {
        Campaign campaign = Campaign.builder()
                .organizerId(organizerId).title(title).description(description)
                .goalAmount(goalAmount).currency(currency).category(category)
                .status("DRAFT").startDate(startDate).endDate(endDate).build();
        return campaignRepository.save(campaign);
    }

    @Transactional
    public Donation donate(Long campaignId, Long donorId, String donorName, String donorEmail,
                            BigDecimal amount, String paymentMethod, boolean anonymous,
                            String message, boolean recurring, String recurringInterval) {
        Campaign campaign = campaignRepository.findById(campaignId)
                .orElseThrow(() -> new RuntimeException("Campaign not found"));
        if (!"ACTIVE".equals(campaign.getStatus())) {
            throw new IllegalStateException("Campaign is not accepting donations");
        }
        if (campaign.getEndDate() != null && campaign.getEndDate().isBefore(LocalDateTime.now())) {
            throw new IllegalStateException("Campaign has ended");
        }

        // Process payment
        String paymentRef = paymentService.charge(amount, paymentMethod, donorEmail);

        Donation donation = Donation.builder()
                .campaignId(campaignId).donorId(donorId)
                .donorName(anonymous ? "Anonymous" : donorName).donorEmail(donorEmail)
                .amount(amount).currency(campaign.getCurrency())
                .isAnonymous(anonymous).message(message)
                .paymentStatus("COMPLETED").paymentReference(paymentRef)
                .paymentMethod(paymentMethod).isRecurring(recurring)
                .recurringInterval(recurringInterval).build();
        donation = donationRepository.save(donation);

        campaign.setRaisedAmount(campaign.getRaisedAmount().add(amount));
        campaign.setDonorCount(campaign.getDonorCount() + 1);
        campaignRepository.save(campaign);

        // Setup recurring
        if (recurring && recurringInterval != null) {
            setupRecurring(donation, campaignId, donorId, donorEmail, amount, recurringInterval);
        }

        // Send receipt
        String receiptNumber = "RCP-" + donation.getId() + "-" + System.currentTimeMillis();
        donation.setTaxReceiptIssued(true);
        donation.setTaxReceiptNumber(receiptNumber);
        donationRepository.save(donation);
        emailService.sendDonationReceipt(donorEmail, donation, campaign);

        return donation;
    }

    private void setupRecurring(Donation original, Long campaignId, Long donorId,
                                  String donorEmail, BigDecimal amount, String interval) {
        LocalDate nextCharge = "MONTHLY".equals(interval) ?
            LocalDate.now().plusMonths(1) : LocalDate.now().plusWeeks(1);
        RecurringDonation recurring = RecurringDonation.builder()
                .originalDonationId(original.getId()).campaignId(campaignId)
                .donorId(donorId).donorEmail(donorEmail).amount(amount)
                .intervalType(interval).nextChargeDate(nextCharge).status("ACTIVE").build();
        recurringRepository.save(recurring);
    }

    @Scheduled(cron = "0 0 9 * * *")
    @Transactional
    public void processRecurringDonations() {
        List<RecurringDonation> dueToday = recurringRepository
                .findByStatusAndNextChargeDateLessThanEqual("ACTIVE", LocalDate.now());
        for (RecurringDonation rd : dueToday) {
            try {
                paymentService.charge(rd.getAmount(), "SAVED_CARD", rd.getDonorEmail());
                Donation newDonation = Donation.builder()
                        .campaignId(rd.getCampaignId()).donorId(rd.getDonorId())
                        .donorEmail(rd.getDonorEmail()).amount(rd.getAmount())
                        .paymentStatus("COMPLETED").isRecurring(true).build();
                donationRepository.save(newDonation);

                Campaign campaign = campaignRepository.findById(rd.getCampaignId()).orElseThrow();
                campaign.setRaisedAmount(campaign.getRaisedAmount().add(rd.getAmount()));
                campaignRepository.save(campaign);

                rd.setTotalCharges(rd.getTotalCharges() + 1);
                rd.setNextChargeDate("MONTHLY".equals(rd.getIntervalType()) ?
                    rd.getNextChargeDate().plusMonths(1) : rd.getNextChargeDate().plusWeeks(1));
                recurringRepository.save(rd);
            } catch (Exception e) {
                rd.setStatus("FAILED");
                recurringRepository.save(rd);
            }
        }
    }

    @Transactional
    public FundDisbursement createDisbursement(Long campaignId, BigDecimal amount,
                                                 String recipientName, String recipientAccount,
                                                 String description, Long processedBy) {
        Campaign campaign = campaignRepository.findById(campaignId).orElseThrow();
        if (amount.compareTo(campaign.getRaisedAmount()) > 0) {
            throw new IllegalArgumentException("Disbursement exceeds raised amount");
        }
        FundDisbursement disbursement = FundDisbursement.builder()
                .campaignId(campaignId).amount(amount)
                .recipientName(recipientName).recipientAccount(recipientAccount)
                .description(description).processedBy(processedBy)
                .status("COMPLETED").processedAt(LocalDateTime.now()).build();
        return disbursementRepository.save(disbursement);
    }

    public CampaignProgress getProgress(Long campaignId) {
        Campaign campaign = campaignRepository.findById(campaignId)
                .orElseThrow(() -> new RuntimeException("Campaign not found"));
        double percentage = campaign.getGoalAmount().compareTo(BigDecimal.ZERO) > 0 ?
            campaign.getRaisedAmount().divide(campaign.getGoalAmount(), 4,
                java.math.RoundingMode.HALF_UP).doubleValue() * 100 : 0.0;
        return new CampaignProgress(campaign.getId(), campaign.getTitle(),
            campaign.getRaisedAmount(), campaign.getGoalAmount(), percentage,
            campaign.getDonorCount(), campaign.getEndDate());
    }

    public record CampaignProgress(Long campaignId, String title, BigDecimal raised,
        BigDecimal goal, double percentage, int donorCount, LocalDateTime endDate) {}
}
```

### Controller

```java
// FundraisingController.java
@RestController
@RequestMapping("/api/v1/fundraising")
@RequiredArgsConstructor
public class FundraisingController {

    private final FundraisingService service;

    @PostMapping("/campaigns")
    public ResponseEntity<Campaign> createCampaign(@RequestBody CreateCampaignRequest request) {
        return ResponseEntity.ok(service.createCampaign(request.organizerId(), request.title(),
            request.description(), request.goalAmount(), request.currency(),
            request.category(), request.startDate(), request.endDate()));
    }

    @GetMapping("/campaigns/{campaignId}/progress")
    public ResponseEntity<FundraisingService.CampaignProgress> getProgress(
            @PathVariable Long campaignId) {
        return ResponseEntity.ok(service.getProgress(campaignId));
    }

    @PostMapping("/campaigns/{campaignId}/donate")
    public ResponseEntity<Donation> donate(@PathVariable Long campaignId,
            @RequestBody DonateRequest request) {
        return ResponseEntity.ok(service.donate(campaignId, request.donorId(),
            request.donorName(), request.donorEmail(), request.amount(),
            request.paymentMethod(), request.anonymous(), request.message(),
            request.recurring(), request.recurringInterval()));
    }

    @PostMapping("/campaigns/{campaignId}/disbursements")
    public ResponseEntity<FundDisbursement> createDisbursement(@PathVariable Long campaignId,
            @RequestBody DisbursementRequest request) {
        return ResponseEntity.ok(service.createDisbursement(campaignId, request.amount(),
            request.recipientName(), request.recipientAccount(),
            request.description(), request.processedBy()));
    }

    record CreateCampaignRequest(Long organizerId, String title, String description,
        BigDecimal goalAmount, String currency, String category,
        java.time.LocalDateTime startDate, java.time.LocalDateTime endDate) {}
    record DonateRequest(Long donorId, String donorName, String donorEmail, BigDecimal amount,
        String paymentMethod, boolean anonymous, String message,
        boolean recurring, String recurringInterval) {}
    record DisbursementRequest(BigDecimal amount, String recipientName,
        String recipientAccount, String description, Long processedBy) {}
}
```

### docker-compose.yml

```yaml
# docker-compose.yml (Donation & Fundraising Platform)
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: fundraising_db
      POSTGRES_USER: fund_user
      POSTGRES_PASSWORD: fund_pass
    ports:
      - "5432:5432"
    volumes:
      - fund_pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

  fundraising-service:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/fundraising_db
      SPRING_DATASOURCE_USERNAME: fund_user
      SPRING_DATASOURCE_PASSWORD: fund_pass
      SPRING_REDIS_HOST: redis
      SPRING_MAIL_HOST: mailhog
      SPRING_MAIL_PORT: 1025
      PAYMENT_GATEWAY_KEY: ${PAYMENT_KEY}
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - mailhog

volumes:
  fund_pg_data:
```

---

*[← Part 112: Food & Health](./part-112-food-health.md) | [Part 114: Education & Learning →](./part-114-education-learning.md)*

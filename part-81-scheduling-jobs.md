# Part 81: Scheduling Jobs
## ขั้นตอนที่ 2841-2880

**ระดับ:** ระดับสูง (Advanced)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้การสร้างและจัดการ Scheduled Tasks ใน Spring Boot อย่างครบวงจร ตั้งแต่การใช้ @Scheduled พื้นฐาน ไปจนถึง Distributed Scheduling ด้วย ShedLock และ Quartz Scheduler สำหรับระบบ Production

---

## 2841-2845: @Scheduled Tasks พื้นฐานและการตั้งค่า

Spring Boot มีกลไกการทำ Task Scheduling ในตัวผ่าน `@Scheduled` annotation ซึ่งใช้งานง่ายและมีประสิทธิภาพสูง เหมาะสำหรับงานที่ต้องรันซ้ำตามเวลาที่กำหนด

### การเปิดใช้งาน Scheduling

```java
// Application.java
package com.example.scheduling;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling  // เปิดใช้งาน Scheduling
public class SchedulingApplication {
    public static void main(String[] args) {
        SpringApplication.run(SchedulingApplication.class, args);
    }
}
```

### ประเภทของ @Scheduled

```java
// ScheduledTaskExample.java
package com.example.scheduling.tasks;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Slf4j
@Component
public class ScheduledTaskExample {

    private static final DateTimeFormatter FORMATTER = 
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    // รันทุก 5 วินาที (fixedRate = เวลาห่างระหว่างการเริ่มต้นของแต่ละ execution)
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        log.info("Fixed Rate Task - executed at: {}", 
            LocalDateTime.now().format(FORMATTER));
    }

    // รันทุก 5 วินาที หลังจาก task ก่อนหน้าเสร็จสิ้น (fixedDelay)
    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() {
        log.info("Fixed Delay Task - executed at: {}", 
            LocalDateTime.now().format(FORMATTER));
    }

    // รันครั้งแรกหลังจาก 1 วินาที แล้วค่อยรันทุก 5 วินาที
    @Scheduled(initialDelay = 1000, fixedRate = 5000)
    public void initialDelayTask() {
        log.info("Initial Delay Task - executed at: {}", 
            LocalDateTime.now().format(FORMATTER));
    }

    // รันตาม Cron Expression (ทุกวันจันทร์-ศุกร์ เวลา 09:00)
    @Scheduled(cron = "0 0 9 * * MON-FRI")
    public void cronTask() {
        log.info("Cron Task - executed at: {}", 
            LocalDateTime.now().format(FORMATTER));
    }

    // รันทุกๆ 1 นาที โดยใช้ค่าจาก application.properties
    @Scheduled(fixedRateString = "${scheduling.task.rate:60000}")
    public void configurableRateTask() {
        log.info("Configurable Rate Task - executed at: {}", 
            LocalDateTime.now().format(FORMATTER));
    }
}
```

### การตั้งค่า Thread Pool สำหรับ Scheduling

```java
// SchedulingConfig.java
package com.example.scheduling.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;

@Configuration
public class SchedulingConfig {

    @Bean
    public ThreadPoolTaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        // จำนวน thread ที่ใช้รัน scheduled tasks
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-task-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(60);
        // Handler สำหรับ exception ที่เกิดใน scheduled task
        scheduler.setErrorHandler(throwable -> {
            // log error แต่ไม่ให้ task หยุดทำงาน
            System.err.println("Error in scheduled task: " + throwable.getMessage());
        });
        return scheduler;
    }
}
```

---

## 2846-2850: Cron Expression Reference

Cron Expression ใน Spring ใช้รูปแบบ 6 fields (ต่างจาก Unix cron ที่ใช้ 5 fields)

### รูปแบบ Cron Expression

```
┌───────────── วินาที (0-59)
│ ┌───────────── นาที (0-59)
│ │ ┌───────────── ชั่วโมง (0-23)
│ │ │ ┌───────────── วันที่ (1-31)
│ │ │ │ ┌───────────── เดือน (1-12 หรือ JAN-DEC)
│ │ │ │ │ ┌───────────── วันในสัปดาห์ (0-7, MON-SUN, โดย 0 และ 7 คือ อาทิตย์)
│ │ │ │ │ │
* * * * * *
```

### ตัวอย่าง Cron Expressions

```java
// CronExpressionExamples.java
package com.example.scheduling.tasks;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class CronExpressionExamples {

    // ทุกวินาที
    @Scheduled(cron = "* * * * * *")
    public void everySecond() {}

    // ทุกนาที
    @Scheduled(cron = "0 * * * * *")
    public void everyMinute() {}

    // ทุกชั่วโมง
    @Scheduled(cron = "0 0 * * * *")
    public void everyHour() {}

    // ทุกวัน เวลาเที่ยงคืน
    @Scheduled(cron = "0 0 0 * * *")
    public void everyMidnight() {}

    // ทุกวันอาทิตย์ เวลา 00:00
    @Scheduled(cron = "0 0 0 * * SUN")
    public void everySundayMidnight() {}

    // วันทำงาน (จันทร์-ศุกร์) เวลา 08:30
    @Scheduled(cron = "0 30 8 * * MON-FRI")
    public void weekdaysMorning() {}

    // วันที่ 1 ของทุกเดือน เวลา 01:00
    @Scheduled(cron = "0 0 1 1 * *")
    public void firstDayOfMonth() {}

    // ทุก 15 นาที
    @Scheduled(cron = "0 0/15 * * * *")
    public void every15Minutes() {}

    // ทุก 6 ชั่วโมง (00:00, 06:00, 12:00, 18:00)
    @Scheduled(cron = "0 0 0/6 * * *")
    public void every6Hours() {}

    // ทุกวันที่ 1 และ 15 ของเดือน เวลา 10:00
    @Scheduled(cron = "0 0 10 1,15 * *")
    public void twiceAMonth() {}

    // ทุกวันในช่วงเวลาทำการ (09:00-17:00) ทุกชั่วโมง
    @Scheduled(cron = "0 0 9-17 * * MON-FRI")
    public void businessHours() {}

    // ใช้ timezone ที่กำหนด
    @Scheduled(cron = "0 0 9 * * *", zone = "Asia/Bangkok")
    public void bangkokTime() {}
}
```

### ตัวช่วย Cron Expression

```java
// CronHelper.java
package com.example.scheduling.util;

import org.springframework.scheduling.support.CronExpression;

import java.time.ZonedDateTime;

public class CronHelper {

    // ตรวจสอบว่า Cron Expression ถูกต้องหรือไม่
    public static boolean isValidCron(String expression) {
        try {
            CronExpression.parse(expression);
            return true;
        } catch (IllegalArgumentException e) {
            return false;
        }
    }

    // คำนวณเวลา execution ถัดไป
    public static ZonedDateTime nextExecution(String cronExpression) {
        CronExpression cron = CronExpression.parse(cronExpression);
        return cron.next(ZonedDateTime.now());
    }

    // แสดงเวลา execution ถัดไป 5 ครั้ง
    public static void printNextExecutions(String cronExpression, int count) {
        CronExpression cron = CronExpression.parse(cronExpression);
        ZonedDateTime current = ZonedDateTime.now();
        
        System.out.println("Next " + count + " executions for: " + cronExpression);
        for (int i = 0; i < count; i++) {
            current = cron.next(current);
            System.out.println("  " + (i + 1) + ": " + current);
        }
    }
}
```

---

## 2851-2855: Distributed Task Scheduling ด้วย ShedLock

ปัญหาหลักของ @Scheduled คือเมื่อ deploy หลาย instance จะทำให้ task รันซ้ำกัน ShedLock ช่วยแก้ปัญหานี้โดยใช้ distributed lock

### การตั้งค่า ShedLock

```xml
<!-- pom.xml -->
<dependencies>
    <!-- ShedLock core -->
    <dependency>
        <groupId>net.javacrumbs.shedlock</groupId>
        <artifactId>shedlock-spring</artifactId>
        <version>5.10.0</version>
    </dependency>
    
    <!-- ShedLock provider สำหรับ PostgreSQL/MySQL -->
    <dependency>
        <groupId>net.javacrumbs.shedlock</groupId>
        <artifactId>shedlock-provider-jdbc-template</artifactId>
        <version>5.10.0</version>
    </dependency>
    
    <!-- ShedLock provider สำหรับ Redis -->
    <dependency>
        <groupId>net.javacrumbs.shedlock</groupId>
        <artifactId>shedlock-provider-redis-spring</artifactId>
        <version>5.10.0</version>
    </dependency>
</dependencies>
```

### สร้าง Database Table สำหรับ ShedLock

```sql
-- V001__create_shedlock_table.sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP(3) NOT NULL,
    locked_at  TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    locked_by  VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

### การ Config ShedLock

```java
// ShedLockConfig.java
package com.example.scheduling.config;

import net.javacrumbs.shedlock.core.LockProvider;
import net.javacrumbs.shedlock.provider.jdbctemplate.JdbcTemplateLockProvider;
import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.scheduling.annotation.EnableScheduling;

import javax.sql.DataSource;

@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")  // lock สูงสุด 10 นาที
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()  // ใช้เวลาจาก DB แทน server time
                .build()
        );
    }
}
```

### การใช้งาน @SchedulerLock

```java
// DistributedScheduledTasks.java
package com.example.scheduling.tasks;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class DistributedScheduledTasks {

    private final ReportService reportService;
    private final DataCleanupService dataCleanupService;

    // Task นี้จะรันเพียง instance เดียวเท่านั้น
    @Scheduled(cron = "0 0 2 * * *")
    @SchedulerLock(
        name = "generateDailyReport",  // ชื่อ lock ต้องไม่ซ้ำกัน
        lockAtLeastFor = "PT5M",       // lock อย่างน้อย 5 นาที (ISO 8601 Duration)
        lockAtMostFor = "PT30M"        // lock สูงสุด 30 นาที
    )
    public void generateDailyReport() {
        log.info("Starting daily report generation...");
        try {
            reportService.generateDailyReport();
            log.info("Daily report generation completed");
        } catch (Exception e) {
            log.error("Failed to generate daily report", e);
            throw e;
        }
    }

    // Cleanup task รัน instance เดียว
    @Scheduled(cron = "0 0 3 * * SUN")
    @SchedulerLock(name = "weeklyDataCleanup", lockAtMostFor = "PT1H")
    public void weeklyDataCleanup() {
        log.info("Starting weekly data cleanup...");
        dataCleanupService.cleanupOldData();
        log.info("Weekly data cleanup completed");
    }
}
```

### ShedLock กับ Redis

```java
// ShedLockRedisConfig.java
package com.example.scheduling.config;

import net.javacrumbs.shedlock.core.LockProvider;
import net.javacrumbs.shedlock.provider.redis.spring.RedisLockProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;

@Configuration
public class ShedLockRedisConfig {

    @Bean
    public LockProvider lockProvider(RedisConnectionFactory connectionFactory) {
        return new RedisLockProvider(connectionFactory, "spring-boot-app");
    }
}
```

---

## 2856-2862: Quartz Scheduler Integration

Quartz เป็น Job Scheduling library ที่มีความสามารถสูงกว่า @Scheduled มาก รองรับ Job Persistence, Clustering, และการ manage jobs แบบ dynamic

### Dependencies และ Configuration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  quartz:
    job-store-type: jdbc              # เก็บ job ใน database
    jdbc:
      initialize-schema: always       # สร้าง schema อัตโนมัติ
    properties:
      org:
        quartz:
          scheduler:
            instanceName: MyScheduler
            instanceId: AUTO          # generate instance ID อัตโนมัติ
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            tablePrefix: QRTZ_
            isClustered: true         # เปิด Clustering mode
            clusterCheckinInterval: 10000  # ตรวจสอบ cluster ทุก 10 วินาที
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool
            threadCount: 10
            threadPriority: 5
```

### สร้าง Quartz Job

```java
// EmailNotificationJob.java
package com.example.scheduling.quartz;

import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@DisallowConcurrentExecution  // ป้องกันไม่ให้รันซ้ำพร้อมกัน
@PersistJobDataAfterExecution  // บันทึก JobDataMap หลัง execute
public class EmailNotificationJob implements Job {

    @Autowired
    private EmailService emailService;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        
        String emailTo = dataMap.getString("emailTo");
        String subject = dataMap.getString("subject");
        String template = dataMap.getString("template");
        
        // นับจำนวนครั้งที่รัน
        int runCount = dataMap.getInt("runCount");
        dataMap.put("runCount", runCount + 1);
        
        log.info("Executing EmailNotificationJob - attempt #{}, to: {}", 
            runCount + 1, emailTo);
        
        try {
            emailService.sendFromTemplate(emailTo, subject, template, dataMap);
            log.info("Email sent successfully to: {}", emailTo);
        } catch (Exception e) {
            log.error("Failed to send email to: {}", emailTo, e);
            // Refire job หลังจาก 5 นาที
            throw new JobExecutionException(e, true);
        }
    }
}
```

### สร้างและ Schedule Jobs แบบ Dynamic

```java
// QuartzJobService.java
package com.example.scheduling.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class QuartzJobService {

    private final Scheduler scheduler;

    // สร้าง Job ที่รันตาม Cron Expression
    public void scheduleCronJob(
        String jobName,
        String jobGroup,
        Class<? extends Job> jobClass,
        String cronExpression,
        Map<String, Object> data
    ) throws SchedulerException {
        
        JobDataMap jobDataMap = new JobDataMap(data);
        
        JobDetail jobDetail = JobBuilder.newJob(jobClass)
            .withIdentity(jobName, jobGroup)
            .withDescription("Cron Job: " + jobName)
            .usingJobData(jobDataMap)
            .storeDurably()  // เก็บ job แม้ไม่มี trigger
            .build();
        
        CronTrigger trigger = TriggerBuilder.newTrigger()
            .forJob(jobDetail)
            .withIdentity(jobName + "_trigger", jobGroup)
            .withSchedule(CronScheduleBuilder
                .cronSchedule(cronExpression)
                .withMisfireHandlingInstructionFireAndProceed()
            )
            .build();
        
        if (scheduler.checkExists(jobDetail.getKey())) {
            scheduler.rescheduleJob(trigger.getKey(), trigger);
            log.info("Rescheduled job: {}/{}", jobGroup, jobName);
        } else {
            scheduler.scheduleJob(jobDetail, trigger);
            log.info("Scheduled new cron job: {}/{}", jobGroup, jobName);
        }
    }

    // สร้าง Job ที่รันครั้งเดียวในอนาคต
    public void scheduleOneTimeJob(
        String jobName,
        String jobGroup,
        Class<? extends Job> jobClass,
        Date runAt,
        Map<String, Object> data
    ) throws SchedulerException {
        
        JobDataMap jobDataMap = new JobDataMap(data);
        
        JobDetail jobDetail = JobBuilder.newJob(jobClass)
            .withIdentity(jobName, jobGroup)
            .usingJobData(jobDataMap)
            .build();
        
        SimpleTrigger trigger = TriggerBuilder.newTrigger()
            .forJob(jobDetail)
            .withIdentity(jobName + "_trigger", jobGroup)
            .startAt(runAt)
            .withSchedule(SimpleScheduleBuilder.simpleSchedule())
            .build();
        
        scheduler.scheduleJob(jobDetail, trigger);
        log.info("Scheduled one-time job: {}/{} at {}", jobGroup, jobName, runAt);
    }

    // หยุด Job ชั่วคราว
    public void pauseJob(String jobName, String jobGroup) throws SchedulerException {
        JobKey jobKey = new JobKey(jobName, jobGroup);
        scheduler.pauseJob(jobKey);
        log.info("Paused job: {}/{}", jobGroup, jobName);
    }

    // เริ่ม Job ต่อ
    public void resumeJob(String jobName, String jobGroup) throws SchedulerException {
        JobKey jobKey = new JobKey(jobName, jobGroup);
        scheduler.resumeJob(jobKey);
        log.info("Resumed job: {}/{}", jobGroup, jobName);
    }

    // ลบ Job
    public void deleteJob(String jobName, String jobGroup) throws SchedulerException {
        JobKey jobKey = new JobKey(jobName, jobGroup);
        scheduler.deleteJob(jobKey);
        log.info("Deleted job: {}/{}", jobGroup, jobName);
    }

    // รัน Job ทันที (manual trigger)
    public void triggerJobNow(String jobName, String jobGroup) throws SchedulerException {
        JobKey jobKey = new JobKey(jobName, jobGroup);
        scheduler.triggerJob(jobKey);
        log.info("Manually triggered job: {}/{}", jobGroup, jobName);
    }
}
```

---

## 2863-2868: Job Persistence และ Clustering

### การตรวจสอบสถานะ Jobs

```java
// JobMonitoringService.java
package com.example.scheduling.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.quartz.impl.matchers.GroupMatcher;
import org.springframework.stereotype.Service;

import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class JobMonitoringService {

    private final Scheduler scheduler;

    // ดึงข้อมูล Jobs ทั้งหมด
    public List<JobInfo> getAllJobs() throws SchedulerException {
        List<JobInfo> jobs = new ArrayList<>();
        
        for (String group : scheduler.getJobGroupNames()) {
            for (JobKey jobKey : scheduler.getJobKeys(GroupMatcher.jobGroupEquals(group))) {
                JobDetail jobDetail = scheduler.getJobDetail(jobKey);
                List<? extends Trigger> triggers = scheduler.getTriggersOfJob(jobKey);
                
                JobInfo info = new JobInfo();
                info.setName(jobKey.getName());
                info.setGroup(jobKey.getGroup());
                info.setDescription(jobDetail.getDescription());
                info.setJobClass(jobDetail.getJobClass().getName());
                
                for (Trigger trigger : triggers) {
                    info.setNextFireTime(trigger.getNextFireTime());
                    info.setPreviousFireTime(trigger.getPreviousFireTime());
                    
                    Trigger.TriggerState state = scheduler.getTriggerState(trigger.getKey());
                    info.setState(state.name());
                    
                    if (trigger instanceof CronTrigger cronTrigger) {
                        info.setCronExpression(cronTrigger.getCronExpression());
                    }
                }
                
                jobs.add(info);
            }
        }
        
        return jobs;
    }

    // ดึงสถานะ Scheduler
    public Map<String, Object> getSchedulerStatus() throws SchedulerException {
        Map<String, Object> status = new HashMap<>();
        SchedulerMetaData metaData = scheduler.getMetaData();
        
        status.put("schedulerName", metaData.getSchedulerName());
        status.put("schedulerInstanceId", metaData.getSchedulerInstanceId());
        status.put("isStarted", metaData.isStarted());
        status.put("isPaused", metaData.isInStandbyMode());
        status.put("isShutdown", metaData.isShutdown());
        status.put("jobsExecuted", metaData.getNumberOfJobsExecuted());
        status.put("threadPoolSize", metaData.getThreadPoolSize());
        status.put("jobStorePersistent", metaData.isJobStoreSupportsPersistence());
        status.put("isClustered", metaData.isJobStoreClustered());
        
        return status;
    }
}
```

### Job Data Transfer Object

```java
// JobInfo.java
package com.example.scheduling.dto;

import lombok.Data;

import java.util.Date;

@Data
public class JobInfo {
    private String name;
    private String group;
    private String description;
    private String jobClass;
    private String cronExpression;
    private Date nextFireTime;
    private Date previousFireTime;
    private String state;
}
```

---

## 2869-2873: Job Monitoring และ Alerting

### Job Execution Listener

```java
// JobExecutionListener.java
package com.example.scheduling.listener;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.time.Instant;

@Slf4j
@Component
@RequiredArgsConstructor
public class JobExecutionListener implements JobListener {

    private final AlertService alertService;
    private final MetricsService metricsService;

    @Override
    public String getName() {
        return "JobExecutionListener";
    }

    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        String jobName = context.getJobDetail().getKey().getName();
        log.info("Job starting: {}", jobName);
        
        // บันทึกเวลาเริ่มต้น
        context.put("startTime", Instant.now());
        
        // บันทึก metrics
        metricsService.incrementJobStarted(jobName);
    }

    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException exception) {
        String jobName = context.getJobDetail().getKey().getName();
        Instant startTime = (Instant) context.get("startTime");
        Duration duration = Duration.between(startTime, Instant.now());
        
        if (exception != null) {
            log.error("Job failed: {} after {}ms", jobName, duration.toMillis(), exception);
            
            // ส่ง alert เมื่อ job ล้มเหลว
            alertService.sendJobFailureAlert(
                jobName,
                exception.getMessage(),
                duration.toMillis()
            );
            
            metricsService.incrementJobFailed(jobName);
        } else {
            log.info("Job completed: {} in {}ms", jobName, duration.toMillis());
            
            // ส่ง alert ถ้า job ใช้เวลานานเกินไป
            if (duration.toMinutes() > 30) {
                alertService.sendSlowJobAlert(jobName, duration);
            }
            
            metricsService.recordJobDuration(jobName, duration.toMillis());
            metricsService.incrementJobSucceeded(jobName);
        }
    }

    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {
        String jobName = context.getJobDetail().getKey().getName();
        log.warn("Job execution was vetoed: {}", jobName);
    }
}
```

### Alert Service

```java
// AlertService.java
package com.example.scheduling.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Slf4j
@Service
@RequiredArgsConstructor
public class AlertService {

    private final JavaMailSender mailSender;
    private final SlackNotificationService slackService;

    public void sendJobFailureAlert(String jobName, String errorMessage, long durationMs) {
        String subject = "[ALERT] Scheduled Job Failed: " + jobName;
        String body = String.format(
            "Job '%s' failed after %d ms\n\nError: %s",
            jobName, durationMs, errorMessage
        );
        
        // ส่งอีเมล
        sendEmail("ops-team@company.com", subject, body);
        
        // ส่ง Slack notification
        slackService.sendMessage(
            "#alerts",
            ":red_circle: *Job Failed*: `" + jobName + "`\n" + errorMessage
        );
    }

    public void sendSlowJobAlert(String jobName, Duration duration) {
        log.warn("Job '{}' took {} minutes to complete", jobName, duration.toMinutes());
        slackService.sendMessage(
            "#monitoring",
            ":warning: *Slow Job*: `" + jobName + "` took " + duration.toMinutes() + " minutes"
        );
    }

    private void sendEmail(String to, String subject, String body) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(to);
            message.setSubject(subject);
            message.setText(body);
            mailSender.send(message);
        } catch (Exception e) {
            log.error("Failed to send alert email", e);
        }
    }
}
```

### Metrics สำหรับ Scheduled Jobs

```java
// SchedulingMetricsConfig.java
package com.example.scheduling.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class MetricsService {

    private final MeterRegistry meterRegistry;

    public MetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void incrementJobStarted(String jobName) {
        Counter.builder("scheduled.job.started")
            .tag("job_name", jobName)
            .register(meterRegistry)
            .increment();
    }

    public void incrementJobSucceeded(String jobName) {
        Counter.builder("scheduled.job.succeeded")
            .tag("job_name", jobName)
            .register(meterRegistry)
            .increment();
    }

    public void incrementJobFailed(String jobName) {
        Counter.builder("scheduled.job.failed")
            .tag("job_name", jobName)
            .register(meterRegistry)
            .increment();
    }

    public void recordJobDuration(String jobName, long durationMs) {
        Timer.builder("scheduled.job.duration")
            .tag("job_name", jobName)
            .register(meterRegistry)
            .record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

---

## 2874-2880: Graceful Shutdown ของ Scheduled Tasks

การหยุด Scheduled Tasks อย่าง graceful เป็นสิ่งสำคัญมากเพื่อป้องกันข้อมูลเสียหาย

### Graceful Shutdown Configuration

```java
// GracefulShutdownConfig.java
package com.example.scheduling.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;

@Slf4j
@Configuration
public class GracefulShutdownConfig {

    @Bean
    public ThreadPoolTaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-");
        
        // รอให้ tasks เสร็จก่อน shutdown
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        
        // รอสูงสุด 120 วินาที
        scheduler.setAwaitTerminationSeconds(120);
        
        scheduler.setErrorHandler(t -> 
            log.error("Unhandled exception in scheduled task", t)
        );
        
        return scheduler;
    }
}
```

### การจัดการ Shutdown Events

```java
// SchedulingLifecycleManager.java
package com.example.scheduling.lifecycle;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.springframework.context.SmartLifecycle;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicBoolean;

@Slf4j
@Component
@RequiredArgsConstructor
public class SchedulingLifecycleManager implements SmartLifecycle {

    private final Scheduler quartzScheduler;
    private final AtomicBoolean running = new AtomicBoolean(false);

    @Override
    public void start() {
        try {
            quartzScheduler.start();
            running.set(true);
            log.info("Quartz Scheduler started");
        } catch (SchedulerException e) {
            throw new RuntimeException("Failed to start Quartz Scheduler", e);
        }
    }

    @Override
    public void stop() {
        log.info("Initiating graceful shutdown of Quartz Scheduler...");
        try {
            // waitForJobsToComplete = true: รอให้ jobs ที่รันอยู่เสร็จก่อน
            quartzScheduler.shutdown(true);
            running.set(false);
            log.info("Quartz Scheduler shutdown completed");
        } catch (SchedulerException e) {
            log.error("Error during Quartz Scheduler shutdown", e);
        }
    }

    @Override
    public boolean isRunning() {
        return running.get();
    }

    @Override
    public int getPhase() {
        // Phase สูงหมายถึง stop ก่อน (default คือ DEFAULT_PHASE = Integer.MAX_VALUE)
        return Integer.MAX_VALUE - 100;
    }
}
```

### Cancellable Tasks

```java
// CancellableTaskService.java
package com.example.scheduling.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ScheduledFuture;

@Slf4j
@Service
public class CancellableTaskService {

    private final TaskScheduler taskScheduler;
    private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();

    public CancellableTaskService(TaskScheduler taskScheduler) {
        this.taskScheduler = taskScheduler;
    }

    // เพิ่ม task ใหม่แบบ dynamic
    public void scheduleTask(String taskId, Runnable task, String cronExpression) {
        // ยกเลิก task เดิมถ้ามีอยู่
        cancelTask(taskId);
        
        ScheduledFuture<?> future = taskScheduler.schedule(
            task,
            new CronTrigger(cronExpression)
        );
        
        scheduledTasks.put(taskId, future);
        log.info("Scheduled task '{}' with cron '{}'", taskId, cronExpression);
    }

    // ยกเลิก task
    public boolean cancelTask(String taskId) {
        ScheduledFuture<?> future = scheduledTasks.remove(taskId);
        if (future != null) {
            boolean cancelled = future.cancel(false);  // false = รอให้ task ปัจจุบันเสร็จ
            log.info("Task '{}' cancelled: {}", taskId, cancelled);
            return cancelled;
        }
        return false;
    }

    // หยุดทุก tasks
    public void cancelAllTasks() {
        log.info("Cancelling all {} scheduled tasks...", scheduledTasks.size());
        scheduledTasks.forEach((id, future) -> {
            future.cancel(false);
            log.info("Cancelled task: {}", id);
        });
        scheduledTasks.clear();
    }
}
```

### REST API สำหรับจัดการ Jobs

```java
// JobManagementController.java
package com.example.scheduling.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/jobs")
@RequiredArgsConstructor
public class JobManagementController {

    private final QuartzJobService jobService;
    private final JobMonitoringService monitoringService;

    @GetMapping
    public ResponseEntity<List<JobInfo>> getAllJobs() throws Exception {
        return ResponseEntity.ok(monitoringService.getAllJobs());
    }

    @GetMapping("/status")
    public ResponseEntity<Map<String, Object>> getSchedulerStatus() throws Exception {
        return ResponseEntity.ok(monitoringService.getSchedulerStatus());
    }

    @PostMapping("/{group}/{name}/trigger")
    public ResponseEntity<String> triggerJob(
        @PathVariable String group,
        @PathVariable String name
    ) throws Exception {
        jobService.triggerJobNow(name, group);
        return ResponseEntity.ok("Job triggered: " + group + "/" + name);
    }

    @PostMapping("/{group}/{name}/pause")
    public ResponseEntity<String> pauseJob(
        @PathVariable String group,
        @PathVariable String name
    ) throws Exception {
        jobService.pauseJob(name, group);
        return ResponseEntity.ok("Job paused: " + group + "/" + name);
    }

    @PostMapping("/{group}/{name}/resume")
    public ResponseEntity<String> resumeJob(
        @PathVariable String group,
        @PathVariable String name
    ) throws Exception {
        jobService.resumeJob(name, group);
        return ResponseEntity.ok("Job resumed: " + group + "/" + name);
    }

    @DeleteMapping("/{group}/{name}")
    public ResponseEntity<String> deleteJob(
        @PathVariable String group,
        @PathVariable String name
    ) throws Exception {
        jobService.deleteJob(name, group);
        return ResponseEntity.ok("Job deleted: " + group + "/" + name);
    }
}
```

---

## สรุป Part 81

ในบทนี้เราได้เรียนรู้:

1. **@Scheduled Tasks** - การใช้ fixedRate, fixedDelay, และ cron expressions
2. **ShedLock** - การป้องกัน duplicate execution ในระบบ distributed
3. **Quartz Scheduler** - Job persistence, clustering, และ dynamic scheduling
4. **Job Monitoring** - การติดตามและ alert เมื่อ jobs ล้มเหลว
5. **Graceful Shutdown** - การหยุด tasks อย่างปลอดภัย

---

*[← Part 80: Real-time Features](./part-80-realtime-features.md) | [Part 82: Rate Limiting →](./part-82-rate-limiting.md)*

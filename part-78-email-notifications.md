# Part 78: Email & Notifications
## ขั้นตอนที่ 2721-2760

**ระดับ:** ระดับสูง (Advanced)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** เรียนรู้การส่ง email ด้วย Spring Mail, สร้าง email template ด้วย Thymeleaf, ส่ง async ผ่าน message queue, Push notification ด้วย Firebase FCM และ SMS ด้วย Twilio

---

## ขั้นตอนที่ 2721: ตั้งค่า Spring Mail

Spring Mail รองรับ SMTP protocol ซึ่งใช้ได้กับ Gmail, AWS SES, SendGrid และอื่น ๆ

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.firebase</groupId>
        <artifactId>firebase-admin</artifactId>
        <version>9.2.0</version>
    </dependency>
    <dependency>
        <groupId>com.twilio.sdk</groupId>
        <artifactId>twilio</artifactId>
        <version>9.14.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
          connectiontimeout: 5000
          timeout: 3000
          writetimeout: 5000
    default-encoding: UTF-8

  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

app:
  mail:
    from: noreply@example.com
    from-name: MyApp
    base-url: https://app.example.com

  firebase:
    credentials-file: classpath:firebase-service-account.json
    project-id: my-firebase-project

  twilio:
    account-sid: ${TWILIO_ACCOUNT_SID}
    auth-token: ${TWILIO_AUTH_TOKEN}
    from-number: ${TWILIO_FROM_NUMBER}
```

---

## ขั้นตอนที่ 2722: Email Service พื้นฐาน

```java
// service/EmailService.java
package com.example.notification.service;

import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.ClassPathResource;
import org.springframework.mail.MailException;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

import java.util.Map;
import java.util.concurrent.CompletableFuture;

@Slf4j
@Service
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;

    @Value("${app.mail.from}")
    private String fromEmail;

    @Value("${app.mail.from-name}")
    private String fromName;

    // ส่ง plain text email
    public void sendSimple(String to, String subject, String text) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(fromName + " <" + fromEmail + ">");
            message.setTo(to);
            message.setSubject(subject);
            message.setText(text);
            mailSender.send(message);
            log.info("Simple email sent to {}", to);
        } catch (MailException e) {
            log.error("Failed to send simple email to {}: {}", to, e.getMessage());
            throw new EmailSendException("ไม่สามารถส่ง email ได้", e);
        }
    }

    // ส่ง HTML email ด้วย Thymeleaf template
    public void sendHtml(String to, String subject, String templateName,
                          Map<String, Object> variables) throws MessagingException {
        Context context = new Context();
        context.setVariables(variables);
        String htmlContent = templateEngine.process(templateName, context);

        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

        helper.setFrom(fromEmail, fromName);
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(htmlContent, true);

        mailSender.send(message);
        log.info("HTML email sent to {} with template {}", to, templateName);
    }

    // ส่ง email พร้อม attachment
    public void sendWithAttachment(String to, String subject, String templateName,
                                    Map<String, Object> variables,
                                    String attachmentName, byte[] attachment,
                                    String attachmentType) throws MessagingException {
        Context context = new Context();
        context.setVariables(variables);
        String htmlContent = templateEngine.process(templateName, context);

        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

        helper.setFrom(fromEmail, fromName);
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(htmlContent, true);
        helper.addAttachment(attachmentName,
            () -> new java.io.ByteArrayInputStream(attachment),
            attachmentType);

        mailSender.send(message);
    }

    // Async email sending
    @Async("emailExecutor")
    public CompletableFuture<Void> sendAsync(String to, String subject,
                                              String templateName, Map<String, Object> vars) {
        try {
            sendHtml(to, subject, templateName, vars);
            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            log.error("Async email failed for {}: {}", to, e.getMessage());
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

---

## ขั้นตอนที่ 2723: Thymeleaf Email Templates

```html
<!-- resources/templates/emails/welcome.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ยินดีต้อนรับ</title>
    <style>
        body {
            font-family: 'Sarabun', Arial, sans-serif;
            background-color: #f5f5f5;
            margin: 0;
            padding: 0;
        }
        .container {
            max-width: 600px;
            margin: 20px auto;
            background-color: #ffffff;
            border-radius: 8px;
            overflow: hidden;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .header {
            background-color: #1a73e8;
            color: white;
            padding: 30px 20px;
            text-align: center;
        }
        .header h1 {
            margin: 0;
            font-size: 24px;
        }
        .content {
            padding: 30px;
        }
        .button {
            display: inline-block;
            background-color: #1a73e8;
            color: white;
            padding: 14px 30px;
            border-radius: 6px;
            text-decoration: none;
            font-size: 16px;
            font-weight: bold;
            margin: 20px 0;
        }
        .footer {
            background-color: #f8f9fa;
            padding: 20px;
            text-align: center;
            font-size: 12px;
            color: #666;
        }
    </style>
</head>
<body>
<div class="container">
    <div class="header">
        <h1>ยินดีต้อนรับ!</h1>
    </div>
    <div class="content">
        <p>สวัสดีคุณ <strong th:text="${username}">User</strong>,</p>
        <p>ขอบคุณที่สมัครสมาชิกกับ MyApp บัญชีของคุณพร้อมใช้งานแล้ว</p>
        <p>กรุณากดปุ่มด้านล่างเพื่อยืนยัน email ของคุณ:</p>

        <div style="text-align: center;">
            <a th:href="${verificationUrl}" class="button">ยืนยัน Email</a>
        </div>

        <p>หรือคัดลอกลิงก์นี้ไปวางในเบราว์เซอร์:</p>
        <p style="word-break: break-all; color: #1a73e8;" th:text="${verificationUrl}"></p>

        <p>ลิงก์นี้จะหมดอายุภายใน <strong th:text="${expiresIn}">24</strong> ชั่วโมง</p>
    </div>
    <div class="footer">
        <p>หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้</p>
        <p>&copy; 2024 MyApp. สงวนสิทธิ์ทั้งหมด</p>
    </div>
</div>
</body>
</html>
```

```html
<!-- resources/templates/emails/order-confirmation.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="th">
<head>
    <meta charset="UTF-8">
    <title>ยืนยันคำสั่งซื้อ</title>
    <style>
        body { font-family: 'Sarabun', Arial, sans-serif; }
        .order-table { width: 100%; border-collapse: collapse; }
        .order-table th, .order-table td {
            padding: 12px;
            border: 1px solid #ddd;
            text-align: left;
        }
        .order-table th { background-color: #f2f2f2; }
        .total-row { font-weight: bold; background-color: #fff3cd; }
    </style>
</head>
<body>
<div style="max-width: 600px; margin: 0 auto; padding: 20px;">
    <h2 style="color: #1a73e8;">ยืนยันคำสั่งซื้อ #<span th:text="${order.orderNumber}"></span></h2>

    <p>สวัสดีคุณ <strong th:text="${customerName}"></strong>,</p>
    <p>เราได้รับคำสั่งซื้อของคุณแล้ว และกำลังดำเนินการ</p>

    <h3>รายการสินค้า</h3>
    <table class="order-table">
        <thead>
        <tr>
            <th>สินค้า</th>
            <th>จำนวน</th>
            <th>ราคา</th>
            <th>รวม</th>
        </tr>
        </thead>
        <tbody>
        <tr th:each="item : ${order.items}">
            <td th:text="${item.productName}"></td>
            <td th:text="${item.quantity}"></td>
            <td th:text="${#numbers.formatDecimal(item.price, 0, 'COMMA', 2, 'POINT')} + ' บาท'"></td>
            <td th:text="${#numbers.formatDecimal(item.subtotal, 0, 'COMMA', 2, 'POINT')} + ' บาท'"></td>
        </tr>
        </tbody>
        <tfoot>
        <tr class="total-row">
            <td colspan="3" style="text-align: right;">ยอดรวม:</td>
            <td th:text="${#numbers.formatDecimal(order.total, 0, 'COMMA', 2, 'POINT')} + ' บาท'"></td>
        </tr>
        </tfoot>
    </table>

    <div style="margin-top: 20px; padding: 15px; background-color: #f8f9fa; border-radius: 6px;">
        <h4>ที่อยู่จัดส่ง</h4>
        <p th:text="${order.shippingAddress}"></p>
        <p>วิธีจัดส่ง: <strong th:text="${order.shippingMethod}"></strong></p>
        <p>คาดว่าจะได้รับ: <strong th:text="${order.estimatedDelivery}"></strong></p>
    </div>
</div>
</body>
</html>
```

---

## ขั้นตอนที่ 2724: Email Queue ด้วย RabbitMQ

การส่ง email ผ่าน queue ช่วยให้ระบบ resilient และ scalable มากขึ้น

```java
// config/RabbitMQConfig.java
package com.example.notification.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String EMAIL_QUEUE = "email.queue";
    public static final String EMAIL_EXCHANGE = "email.exchange";
    public static final String EMAIL_ROUTING_KEY = "email.send";
    public static final String EMAIL_DLQ = "email.deadletter";

    @Bean
    Queue emailQueue() {
        return QueueBuilder.durable(EMAIL_QUEUE)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", EMAIL_DLQ)
            .withArgument("x-message-ttl", 86400000) // 24 ชั่วโมง
            .build();
    }

    @Bean
    Queue emailDeadLetterQueue() {
        return QueueBuilder.durable(EMAIL_DLQ).build();
    }

    @Bean
    DirectExchange emailExchange() {
        return new DirectExchange(EMAIL_EXCHANGE);
    }

    @Bean
    Binding emailBinding(Queue emailQueue, DirectExchange emailExchange) {
        return BindingBuilder.bind(emailQueue)
            .to(emailExchange)
            .with(EMAIL_ROUTING_KEY);
    }

    @Bean
    Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                   Jackson2JsonMessageConverter converter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(converter);
        return template;
    }
}
```

```java
// service/EmailQueueService.java
package com.example.notification.service;

import com.example.notification.config.RabbitMQConfig;
import com.example.notification.dto.EmailMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;

import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class EmailQueueService {

    private final RabbitTemplate rabbitTemplate;
    private final EmailService emailService;

    // ส่ง email message ไปยัง queue
    public void queueEmail(String to, String subject, String template,
                            Map<String, Object> variables) {
        EmailMessage message = new EmailMessage(to, subject, template, variables);
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.EMAIL_EXCHANGE,
            RabbitMQConfig.EMAIL_ROUTING_KEY,
            message
        );
        log.info("Queued email to {} with template {}", to, template);
    }

    // Consume จาก queue
    @RabbitListener(queues = RabbitMQConfig.EMAIL_QUEUE)
    @Retryable(
        retryFor = Exception.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 5000, multiplier = 2)
    )
    public void processEmailMessage(EmailMessage message) {
        log.info("Processing email to {}", message.to());
        try {
            emailService.sendHtml(
                message.to(),
                message.subject(),
                message.templateName(),
                message.variables()
            );
            log.info("Email sent successfully to {}", message.to());
        } catch (Exception e) {
            log.error("Failed to process email to {}: {}", message.to(), e.getMessage());
            throw new RuntimeException("Email processing failed", e);
        }
    }

    // Consume จาก dead letter queue เพื่อ log / alert
    @RabbitListener(queues = RabbitMQConfig.EMAIL_DLQ)
    public void handleDeadLetterEmail(EmailMessage message) {
        log.error("Email permanently failed for {}: subject={}",
            message.to(), message.subject());
        // ส่ง alert ไปยัง monitoring system
        // alertService.sendAlert(...)
    }
}
```

```java
// dto/EmailMessage.java
package com.example.notification.dto;

import java.util.Map;

public record EmailMessage(
    String to,
    String subject,
    String templateName,
    Map<String, Object> variables
) {}
```

---

## ขั้นตอนที่ 2725: Firebase Push Notification

```java
// config/FirebaseConfig.java
package com.example.notification.config;

import com.google.auth.oauth2.GoogleCredentials;
import com.google.firebase.FirebaseApp;
import com.google.firebase.FirebaseOptions;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;

import java.io.IOException;

@Slf4j
@Configuration
public class FirebaseConfig {

    @Value("${app.firebase.credentials-file}")
    private Resource credentialsFile;

    @Value("${app.firebase.project-id}")
    private String projectId;

    @Bean
    public FirebaseApp firebaseApp() throws IOException {
        if (FirebaseApp.getApps().isEmpty()) {
            GoogleCredentials credentials =
                GoogleCredentials.fromStream(credentialsFile.getInputStream());

            FirebaseOptions options = FirebaseOptions.builder()
                .setCredentials(credentials)
                .setProjectId(projectId)
                .build();

            FirebaseApp app = FirebaseApp.initializeApp(options);
            log.info("Firebase App initialized: {}", app.getName());
            return app;
        }
        return FirebaseApp.getInstance();
    }
}
```

```java
// service/PushNotificationService.java
package com.example.notification.service;

import com.google.firebase.messaging.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class PushNotificationService {

    private final FirebaseMessaging firebaseMessaging;

    // ส่ง notification ไปยัง device เดียว
    public String sendToDevice(String deviceToken, String title, String body,
                                Map<String, String> data) throws FirebaseMessagingException {
        Message message = Message.builder()
            .setToken(deviceToken)
            .setNotification(Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build())
            .putAllData(data != null ? data : Map.of())
            .setAndroidConfig(AndroidConfig.builder()
                .setPriority(AndroidConfig.Priority.HIGH)
                .setNotification(AndroidNotification.builder()
                    .setClickAction("FLUTTER_NOTIFICATION_CLICK")
                    .setChannelId("default")
                    .build())
                .build())
            .setApnsConfig(ApnsConfig.builder()
                .setAps(Aps.builder()
                    .setAlert(ApsAlert.builder()
                        .setTitle(title)
                        .setBody(body)
                        .build())
                    .setSound("default")
                    .setBadge(1)
                    .build())
                .build())
            .build();

        String messageId = firebaseMessaging.send(message);
        log.info("Push notification sent: {}", messageId);
        return messageId;
    }

    // ส่ง notification ไปยังหลาย device พร้อมกัน (batch)
    public BatchResponse sendToMultipleDevices(List<String> tokens, String title,
                                                String body, Map<String, String> data)
            throws FirebaseMessagingException {
        MulticastMessage message = MulticastMessage.builder()
            .addAllTokens(tokens)
            .setNotification(Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build())
            .putAllData(data != null ? data : Map.of())
            .build();

        BatchResponse response = firebaseMessaging.sendEachForMulticast(message);
        log.info("Multicast sent: {} success, {} failure",
            response.getSuccessCount(), response.getFailureCount());

        return response;
    }

    // ส่งไปยัง Topic (group ของ devices)
    public String sendToTopic(String topic, String title, String body,
                               Map<String, String> data) throws FirebaseMessagingException {
        Message message = Message.builder()
            .setTopic(topic)
            .setNotification(Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build())
            .putAllData(data != null ? data : Map.of())
            .build();

        return firebaseMessaging.send(message);
    }

    // Subscribe devices ไปยัง topic
    public void subscribeToTopic(List<String> tokens, String topic)
            throws FirebaseMessagingException {
        firebaseMessaging.subscribeToTopic(tokens, topic);
        log.info("Subscribed {} devices to topic {}", tokens.size(), topic);
    }

    // Unsubscribe จาก topic
    public void unsubscribeFromTopic(List<String> tokens, String topic)
            throws FirebaseMessagingException {
        firebaseMessaging.unsubscribeFromTopic(tokens, topic);
    }

    // ส่ง data-only message (silent notification)
    public String sendDataMessage(String deviceToken, Map<String, String> data)
            throws FirebaseMessagingException {
        Message message = Message.builder()
            .setToken(deviceToken)
            .putAllData(data)
            .setAndroidConfig(AndroidConfig.builder()
                .setPriority(AndroidConfig.Priority.HIGH)
                .build())
            .build();

        return firebaseMessaging.send(message);
    }
}
```

---

## ขั้นตอนที่ 2726: SMS ด้วย Twilio

```java
// config/TwilioConfig.java
package com.example.notification.config;

import com.twilio.Twilio;
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TwilioConfig {

    @Value("${app.twilio.account-sid}")
    private String accountSid;

    @Value("${app.twilio.auth-token}")
    private String authToken;

    @PostConstruct
    public void initTwilio() {
        Twilio.init(accountSid, authToken);
    }
}
```

```java
// service/SmsService.java
package com.example.notification.service;

import com.twilio.rest.api.v2010.account.Message;
import com.twilio.type.PhoneNumber;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class SmsService {

    @Value("${app.twilio.from-number}")
    private String fromNumber;

    // ส่ง SMS ธรรมดา
    public String sendSms(String toNumber, String messageBody) {
        try {
            Message message = Message.creator(
                new PhoneNumber(toNumber),
                new PhoneNumber(fromNumber),
                messageBody
            ).create();

            log.info("SMS sent to {}: SID={}", toNumber, message.getSid());
            return message.getSid();
        } catch (Exception e) {
            log.error("Failed to send SMS to {}: {}", toNumber, e.getMessage());
            throw new RuntimeException("SMS sending failed", e);
        }
    }

    // ส่ง OTP
    public String sendOtp(String toNumber, String otp) {
        String message = String.format(
            "รหัส OTP ของคุณคือ: %s\nรหัสนี้จะหมดอายุใน 5 นาที\nอย่าแชร์รหัสนี้กับใคร",
            otp
        );
        return sendSms(toNumber, message);
    }

    // ส่ง notification สั้น
    public String sendOrderUpdate(String toNumber, String orderNumber, String status) {
        String message = String.format(
            "คำสั่งซื้อ #%s ของคุณ%s\nดูรายละเอียดที่ app.example.com",
            orderNumber, status
        );
        return sendSms(toNumber, message);
    }
}
```

---

## ขั้นตอนที่ 2727: Notification Service รวม

```java
// service/NotificationService.java
package com.example.notification.service;

import com.example.notification.entity.NotificationRecord;
import com.example.notification.entity.UserNotificationPreference;
import com.example.notification.repository.NotificationRecordRepository;
import com.example.notification.repository.UserPreferenceRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final EmailQueueService emailQueueService;
    private final PushNotificationService pushService;
    private final SmsService smsService;
    private final NotificationRecordRepository recordRepository;
    private final UserPreferenceRepository preferenceRepository;

    // ส่ง notification ตาม preference ของ user
    @Async
    public void notify(String userId, NotificationEvent event) {
        UserNotificationPreference prefs = preferenceRepository
            .findByUserId(userId)
            .orElse(UserNotificationPreference.defaultPreferences(userId));

        // ส่ง Email
        if (prefs.isEmailEnabled() && event.getEmailTemplate() != null) {
            try {
                emailQueueService.queueEmail(
                    event.getUserEmail(),
                    event.getEmailSubject(),
                    event.getEmailTemplate(),
                    event.getTemplateVariables()
                );
                recordNotification(userId, "EMAIL", event.getType(), "QUEUED");
            } catch (Exception e) {
                log.error("Failed to queue email notification: {}", e.getMessage());
                recordNotification(userId, "EMAIL", event.getType(), "FAILED");
            }
        }

        // ส่ง Push Notification
        if (prefs.isPushEnabled() && event.getPushTitle() != null) {
            try {
                if (prefs.getDeviceTokens() != null && !prefs.getDeviceTokens().isEmpty()) {
                    pushService.sendToMultipleDevices(
                        prefs.getDeviceTokens(),
                        event.getPushTitle(),
                        event.getPushBody(),
                        event.getPushData()
                    );
                    recordNotification(userId, "PUSH", event.getType(), "SENT");
                }
            } catch (Exception e) {
                log.error("Failed to send push notification: {}", e.getMessage());
                recordNotification(userId, "PUSH", event.getType(), "FAILED");
            }
        }

        // ส่ง SMS
        if (prefs.isSmsEnabled() && event.getSmsMessage() != null &&
                prefs.getPhoneNumber() != null) {
            try {
                smsService.sendSms(prefs.getPhoneNumber(), event.getSmsMessage());
                recordNotification(userId, "SMS", event.getType(), "SENT");
            } catch (Exception e) {
                log.error("Failed to send SMS notification: {}", e.getMessage());
                recordNotification(userId, "SMS", event.getType(), "FAILED");
            }
        }
    }

    private void recordNotification(String userId, String channel,
                                     String type, String status) {
        NotificationRecord record = NotificationRecord.builder()
            .userId(userId)
            .channel(channel)
            .notificationType(type)
            .status(status)
            .sentAt(java.time.LocalDateTime.now())
            .build();
        recordRepository.save(record);
    }
}
```

---

## ขั้นตอนที่ 2728: NotificationEvent และ Entity

```java
// event/NotificationEvent.java
package com.example.notification.event;

import lombok.Builder;
import lombok.Data;
import java.util.Map;
import java.util.List;

@Data
@Builder
public class NotificationEvent {
    private String userId;
    private String type;  // ORDER_PLACED, PAYMENT_SUCCESS, etc.
    private String userEmail;

    // Email
    private String emailSubject;
    private String emailTemplate;
    private Map<String, Object> templateVariables;

    // Push
    private String pushTitle;
    private String pushBody;
    private Map<String, String> pushData;

    // SMS
    private String smsMessage;

    // Factory methods สำหรับ event type ต่าง ๆ
    public static NotificationEvent orderPlaced(String userId, String email,
                                                  String orderNumber) {
        return NotificationEvent.builder()
            .userId(userId)
            .type("ORDER_PLACED")
            .userEmail(email)
            .emailSubject("ยืนยันคำสั่งซื้อ #" + orderNumber)
            .emailTemplate("emails/order-confirmation")
            .templateVariables(Map.of("orderNumber", orderNumber))
            .pushTitle("คำสั่งซื้อสำเร็จ!")
            .pushBody("คำสั่งซื้อ #" + orderNumber + " ของคุณถูกรับแล้ว")
            .pushData(Map.of("screen", "ORDER_DETAIL", "orderId", orderNumber))
            .smsMessage("คำสั่งซื้อ #" + orderNumber + " ของคุณถูกรับแล้ว")
            .build();
    }

    public static NotificationEvent passwordReset(String userId, String email,
                                                    String resetUrl) {
        return NotificationEvent.builder()
            .userId(userId)
            .type("PASSWORD_RESET")
            .userEmail(email)
            .emailSubject("รีเซ็ตรหัสผ่าน")
            .emailTemplate("emails/password-reset")
            .templateVariables(Map.of("resetUrl", resetUrl, "expiresIn", "1"))
            .build();
    }
}
```

```java
// entity/NotificationRecord.java
package com.example.notification.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "notification_records")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NotificationRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private String userId;

    @Column(nullable = false)
    private String channel; // EMAIL, PUSH, SMS

    @Column(name = "notification_type", nullable = false)
    private String notificationType;

    @Column(nullable = false)
    private String status; // QUEUED, SENT, FAILED

    @Column(name = "sent_at")
    private LocalDateTime sentAt;

    @Column(name = "error_message")
    private String errorMessage;
}
```

---

## ขั้นตอนที่ 2729: Async Configuration

```java
// config/AsyncConfig.java
package com.example.notification.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("email-async-");
        executor.setRejectedExecutionHandler(new java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Bean("notificationExecutor")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("notification-");
        executor.initialize();
        return executor;
    }
}
```

---

## ขั้นตอนที่ 2730: ทดสอบ Notification Service

```java
// test/NotificationServiceTest.java
package com.example.notification.service;

import com.example.notification.event.NotificationEvent;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class NotificationServiceTest {

    @Mock
    private EmailQueueService emailQueueService;

    @Mock
    private PushNotificationService pushService;

    @Mock
    private SmsService smsService;

    @InjectMocks
    private NotificationService notificationService;

    @Test
    void shouldSendOrderPlacedNotification() {
        // Given
        NotificationEvent event = NotificationEvent.orderPlaced(
            "user-1", "test@example.com", "ORD-001"
        );

        // When
        notificationService.notify("user-1", event);

        // Then
        verify(emailQueueService).queueEmail(
            eq("test@example.com"),
            contains("ORD-001"),
            eq("emails/order-confirmation"),
            anyMap()
        );
    }

    @Test
    void shouldHandleEmailFailureGracefully() throws Exception {
        // Given
        NotificationEvent event = NotificationEvent.orderPlaced(
            "user-1", "test@example.com", "ORD-001"
        );

        doThrow(new RuntimeException("Queue down"))
            .when(emailQueueService).queueEmail(any(), any(), any(), any());

        // When - ไม่ควร throw exception ออกมา
        notificationService.notify("user-1", event);

        // Then - push ยังส่งได้ปกติ
        verify(pushService).sendToMultipleDevices(any(), any(), any(), any());
    }
}
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:

1. **Spring Mail** - การตั้งค่าและส่ง email แบบ plain text และ HTML
2. **Thymeleaf Templates** - สร้าง email template ที่สวยงามสำหรับภาษาไทย
3. **Async Email** - ส่ง email แบบ async เพื่อไม่ block request
4. **RabbitMQ Queue** - Email queue พร้อม Dead Letter Queue
5. **Firebase FCM** - Push notification สำหรับ iOS และ Android
6. **Twilio SMS** - ส่ง SMS และ OTP
7. **Notification Service** - รวมทุก channel ให้ส่งตาม user preference

---

*[← Part 77: File Storage](./part-77-file-storage.md) | [Part 79: Payment Integration →](./part-79-payment-integration.md)*

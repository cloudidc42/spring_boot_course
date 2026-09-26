# Part 19: Email Service
## ขั้นตอนที่ 491-515

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 3-4 ชั่วโมง  
> **เป้าหมาย:** ส่ง Email ด้วย Template และ Async

---

## ขั้นตอนที่ 491: Email Dependencies

```xml
<!-- Spring Mail -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>

<!-- Thymeleaf Template Engine -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
    <groupId>nz.net.ultraq.thymeleaf</groupId>
    <artifactId>thymeleaf-layout-dialect</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 492: Email Configuration

```yaml
# application.yml
spring:
  mail:
    host: ${MAIL_HOST:smtp.gmail.com}
    port: ${MAIL_PORT:587}
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
        debug: false

app:
  mail:
    from: noreply@myapp.com
    from-name: My App
    base-url: ${APP_BASE_URL:http://localhost:8080}
```

```java
@Configuration
@ConfigurationProperties(prefix = "app.mail")
@Getter @Setter
public class MailConfig {
    private String from;
    private String fromName;
    private String baseUrl;
}
```

---

## ขั้นตอนที่ 493: Email Service Interface

```java
public interface EmailService {
    void sendWelcomeEmail(String to, String username);
    void sendPasswordResetEmail(String to, String resetToken);
    void sendEmailVerificationEmail(String to, String verificationToken);
    void sendOrderConfirmationEmail(String to, OrderResponse order);
    void sendEmail(EmailRequest request);
}

public record EmailRequest(
    String to,
    String subject,
    String templateName,
    Map<String, Object> variables
) {}
```

---

## ขั้นตอนที่ 494: Email Service Implementation

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EmailServiceImpl implements EmailService {
    
    private final JavaMailSender mailSender;
    private final SpringTemplateEngine templateEngine;
    private final MailConfig mailConfig;
    
    @Override
    @Async
    public void sendWelcomeEmail(String to, String username) {
        Map<String, Object> vars = new HashMap<>();
        vars.put("username", username);
        vars.put("loginUrl", mailConfig.getBaseUrl() + "/login");
        
        send(EmailRequest.builder()
            .to(to)
            .subject("Welcome to My App!")
            .templateName("welcome")
            .variables(vars)
            .build());
    }
    
    @Override
    @Async
    public void sendPasswordResetEmail(String to, String resetToken) {
        String resetUrl = mailConfig.getBaseUrl() + "/reset-password?token=" + resetToken;
        
        Map<String, Object> vars = new HashMap<>();
        vars.put("resetUrl", resetUrl);
        vars.put("expiryHours", 1);
        
        send(EmailRequest.builder()
            .to(to)
            .subject("Password Reset Request")
            .templateName("password-reset")
            .variables(vars)
            .build());
    }
    
    @Override
    @Async
    public void sendOrderConfirmationEmail(String to, OrderResponse order) {
        Map<String, Object> vars = new HashMap<>();
        vars.put("order", order);
        vars.put("orderUrl", mailConfig.getBaseUrl() + "/orders/" + order.id());
        
        send(EmailRequest.builder()
            .to(to)
            .subject("Order Confirmed #" + order.orderNumber())
            .templateName("order-confirmation")
            .variables(vars)
            .build());
    }
    
    @Override
    @Async
    public void sendEmail(EmailRequest request) {
        send(request);
    }
    
    private void send(EmailRequest request) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
            
            helper.setFrom(mailConfig.getFrom(), mailConfig.getFromName());
            helper.setTo(request.to());
            helper.setSubject(request.subject());
            
            // Process Thymeleaf template
            Context context = new Context();
            context.setVariables(request.variables());
            context.setVariable("baseUrl", mailConfig.getBaseUrl());
            context.setVariable("currentYear", LocalDate.now().getYear());
            
            String htmlContent = templateEngine.process("email/" + request.templateName(), context);
            helper.setText(htmlContent, true);
            
            mailSender.send(message);
            log.info("Email sent to: {} (subject: {})", request.to(), request.subject());
            
        } catch (Exception e) {
            log.error("Failed to send email to {}: {}", request.to(), e.getMessage(), e);
            throw new EmailException("Failed to send email: " + e.getMessage());
        }
    }
}
```

---

## ขั้นตอนที่ 495: Email Templates (Thymeleaf)

```html
<!-- src/main/resources/templates/email/layout.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title layout:title-pattern="$CONTENT_TITLE">My App</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; background: #f5f5f5; }
        .container { max-width: 600px; margin: 20px auto; background: white; border-radius: 8px; overflow: hidden; }
        .header { background: #2563eb; color: white; padding: 24px; text-align: center; }
        .content { padding: 32px; }
        .button { display: inline-block; background: #2563eb; color: white; padding: 12px 24px; border-radius: 6px; text-decoration: none; }
        .footer { background: #f9fafb; padding: 16px; text-align: center; color: #6b7280; font-size: 12px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>My App</h1>
        </div>
        <div class="content" layout:fragment="content">
            <!-- Content goes here -->
        </div>
        <div class="footer">
            <p>© <span th:text="${currentYear}">2026</span> My App. All rights reserved.</p>
            <p><a th:href="${baseUrl}">Visit our website</a></p>
        </div>
    </div>
</body>
</html>
```

```html
<!-- templates/email/welcome.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{email/layout}">
<head><title>Welcome!</title></head>
<body>
<div layout:fragment="content">
    <h2>Welcome, <span th:text="${username}">User</span>!</h2>
    <p>We're excited to have you on board. Your account has been created successfully.</p>
    <p>
        <a th:href="${loginUrl}" class="button">Login to Your Account</a>
    </p>
    <p>If you have any questions, feel free to contact our support team.</p>
</div>
</body>
</html>
```

```html
<!-- templates/email/password-reset.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{email/layout}">
<body>
<div layout:fragment="content">
    <h2>Password Reset Request</h2>
    <p>We received a request to reset your password. Click the button below to set a new password.</p>
    <p>
        <a th:href="${resetUrl}" class="button">Reset Password</a>
    </p>
    <p>This link will expire in <strong th:text="${expiryHours}">1</strong> hour(s).</p>
    <p style="color: #ef4444;">If you did not request a password reset, please ignore this email.</p>
    <p>Or copy this link: <code th:text="${resetUrl}"></code></p>
</div>
</body>
</html>
```

```html
<!-- templates/email/order-confirmation.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{email/layout}">
<body>
<div layout:fragment="content">
    <h2>Order Confirmed!</h2>
    <p>Thank you for your order. Here's your order summary:</p>
    
    <table style="width: 100%; border-collapse: collapse;">
        <tr style="background: #f3f4f6;">
            <th style="padding: 8px; text-align: left;">Product</th>
            <th style="padding: 8px; text-align: right;">Qty</th>
            <th style="padding: 8px; text-align: right;">Price</th>
        </tr>
        <tr th:each="item : ${order.items}">
            <td style="padding: 8px;" th:text="${item.productName}">Product</td>
            <td style="padding: 8px; text-align: right;" th:text="${item.quantity}">1</td>
            <td style="padding: 8px; text-align: right;" th:text="${'฿' + #numbers.formatDecimal(item.subtotal, 0, 'COMMA', 2, 'POINT')}">฿0.00</td>
        </tr>
        <tr style="border-top: 2px solid #e5e7eb; font-weight: bold;">
            <td colspan="2" style="padding: 8px;">Total</td>
            <td style="padding: 8px; text-align: right;" th:text="${'฿' + #numbers.formatDecimal(order.totalAmount, 0, 'COMMA', 2, 'POINT')}">฿0.00</td>
        </tr>
    </table>
    
    <p>
        <a th:href="${orderUrl}" class="button">View Order Details</a>
    </p>
</div>
</body>
</html>
```

---

## ขั้นตอนที่ 496: Async Email Configuration

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "emailExecutor")
    public TaskExecutor emailTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// Use specific executor for email
@Async("emailExecutor")
public void sendWelcomeEmail(String to, String username) {
    // ...
}
```

---

## ขั้นตอนที่ 497: Email Queue with Events

```java
// Event-driven email sending
@Component
@RequiredArgsConstructor
public class EmailEventListener {
    
    private final EmailService emailService;
    
    @EventListener
    @Async
    public void onUserRegistered(UserRegisteredEvent event) {
        emailService.sendWelcomeEmail(event.email(), event.username());
    }
    
    @EventListener
    @Async
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendOrderConfirmationEmail(event.userEmail(), event.order());
    }
    
    @EventListener
    @Async
    public void onPasswordResetRequested(PasswordResetRequestedEvent event) {
        emailService.sendPasswordResetEmail(event.email(), event.token());
    }
}

// Events
public record UserRegisteredEvent(String email, String username) {}
public record OrderCreatedEvent(String userEmail, OrderResponse order) {}
public record PasswordResetRequestedEvent(String email, String token) {}
```

---

## ขั้นตอนที่ 498: Testing Email

```java
@SpringBootTest
class EmailServiceTest {
    
    @Autowired
    private EmailService emailService;
    
    @MockBean
    private JavaMailSender mailSender;
    
    @Test
    void sendWelcomeEmail_ShouldSendEmail() throws MessagingException {
        // Arrange
        MimeMessage mockMessage = mock(MimeMessage.class);
        when(mailSender.createMimeMessage()).thenReturn(mockMessage);
        
        // Act
        emailService.sendWelcomeEmail("test@test.com", "testuser");
        
        // Assert (need to wait for async)
        await().atMost(2, TimeUnit.SECONDS).untilAsserted(() -> {
            verify(mailSender).send(any(MimeMessage.class));
        });
    }
}
```

---

## ขั้นตอนที่ 499-515: Email Best Practices

```java
// ✅ Retry failed emails
@Retryable(
    value = {MailException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public void sendWithRetry(EmailRequest request) {
    send(request);
}

// ✅ Email delivery tracking
@Entity
public class EmailLog extends BaseEntity {
    private String toEmail;
    private String subject;
    private String status;  // SENT, FAILED, BOUNCED
    private String errorMessage;
    private LocalDateTime sentAt;
}

// ✅ Unsubscribe link in every email
// ✅ DKIM, SPF, DMARC setup for deliverability
// ✅ Use email service (SendGrid, SES) in production
// ✅ Monitor bounce rates and spam complaints

// SendGrid Integration
@Profile("sendgrid")
@Service
public class SendGridEmailService implements EmailService {
    
    @Value("${SENDGRID_API_KEY}")
    private String apiKey;
    
    @Override
    public void sendEmail(EmailRequest request) {
        SendGrid sg = new SendGrid(apiKey);
        Request sgRequest = new Request();
        
        try {
            Mail mail = new Mail(
                new Email(mailConfig.getFrom()),
                request.subject(),
                new Email(request.to()),
                new Content("text/html", renderTemplate(request))
            );
            
            sgRequest.setMethod(Method.POST);
            sgRequest.setEndpoint("mail/send");
            sgRequest.setBody(mail.build());
            
            Response response = sg.api(sgRequest);
            log.info("SendGrid response: {}", response.getStatusCode());
            
        } catch (IOException e) {
            throw new EmailException("SendGrid send failed", e);
        }
    }
}
```

---

*[← Part 18: File Upload](./part-18-file-upload.md) | [Part 20: Pagination Advanced →](./part-20-pagination-advanced.md)*

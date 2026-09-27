# Part 84: Internationalization (i18n)
## ขั้นตอนที่ 2961-3000

**ระดับ:** ระดับสูง (Advanced)
**เวลาเรียน:** 5-7 ชั่วโมง
**เป้าหมาย:** เรียนรู้การสร้างระบบ Internationalization (i18n) ที่ครบวงจร รองรับหลายภาษา หลาย locale พร้อมการจัดการ currency, date format, และ validation messages ในแต่ละภาษา

---

## 2961-2966: MessageSource และ messages.properties

i18n (Internationalization) คือการออกแบบระบบที่รองรับหลายภาษาและวัฒนธรรม Spring Boot มีกลไก MessageSource ที่ใช้งานได้ง่าย

### การตั้งค่า MessageSource

```java
// I18nConfig.java
package com.example.i18n.config;

import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;

import java.nio.charset.StandardCharsets;

@Configuration
public class I18nConfig implements WebMvcConfigurer {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource source = 
            new ReloadableResourceBundleMessageSource();
        
        // ระบุ base names ของ message files
        source.setBasenames(
            "classpath:messages/messages",      // messages.properties, messages_th.properties
            "classpath:messages/errors",        // errors.properties
            "classpath:messages/validation"     // validation.properties
        );
        
        source.setDefaultEncoding(StandardCharsets.UTF_8.name());
        source.setCacheSeconds(3600);           // Cache 1 ชั่วโมง
        source.setUseCodeAsDefaultMessage(true); // ใช้ code เป็น default ถ้าไม่มี message
        source.setFallbackToSystemLocale(false); // ไม่ fallback ไป system locale
        
        return source;
    }

    // ให้ Bean Validation ใช้ MessageSource
    @Bean
    public LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean validator = new LocalValidatorFactoryBean();
        validator.setValidationMessageSource(messageSource());
        return validator;
    }

    // Interceptor เพื่อเปลี่ยนภาษาจาก query parameter
    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");  // ?lang=th
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
```

### Message Files

```properties
# src/main/resources/messages/messages.properties (ภาษาอังกฤษ - default)
app.name=My Spring Boot Application
app.version=1.0.0

welcome.message=Welcome, {0}!
greeting.morning=Good morning, {0}!
greeting.afternoon=Good afternoon, {0}!
greeting.evening=Good evening, {0}!

user.not.found=User with ID {0} not found
user.email.exists=Email {0} is already registered
user.created.success=User {0} has been created successfully

order.created=Order #{0} has been created
order.status.pending=Pending
order.status.processing=Processing
order.status.shipped=Shipped
order.status.delivered=Delivered
order.status.cancelled=Cancelled

product.out.of.stock=Product {0} is out of stock
product.price.label=Price
product.quantity.label=Quantity
```

```properties
# src/main/resources/messages/messages_th.properties (ภาษาไทย)
app.name=แอปพลิเคชัน Spring Boot ของฉัน
app.version=1.0.0

welcome.message=ยินดีต้อนรับ, {0}!
greeting.morning=อรุณสวัสดิ์, {0}!
greeting.afternoon=สวัสดีตอนบ่าย, {0}!
greeting.evening=สวัสดีตอนเย็น, {0}!

user.not.found=ไม่พบผู้ใช้ที่มี ID {0}
user.email.exists=อีเมล {0} ถูกใช้งานแล้ว
user.created.success=สร้างผู้ใช้ {0} สำเร็จแล้ว

order.created=สร้างคำสั่งซื้อ #{0} เรียบร้อยแล้ว
order.status.pending=รอดำเนินการ
order.status.processing=กำลังดำเนินการ
order.status.shipped=จัดส่งแล้ว
order.status.delivered=ได้รับแล้ว
order.status.cancelled=ยกเลิกแล้ว

product.out.of.stock=สินค้า {0} หมด
product.price.label=ราคา
product.quantity.label=จำนวน
```

```properties
# src/main/resources/messages/messages_ja.properties (ภาษาญี่ปุ่น)
app.name=Spring Bootアプリケーション
welcome.message={0}さん、ようこそ！
user.not.found=ID {0} のユーザーが見つかりません
order.created=注文 #{0} が作成されました
```

```properties
# src/main/resources/messages/messages_zh_CN.properties (ภาษาจีน)
app.name=Spring Boot应用程序
welcome.message=欢迎, {0}!
user.not.found=未找到ID为{0}的用户
order.created=订单#{0}已创建
```

### Validation Messages

```properties
# src/main/resources/messages/validation.properties (อังกฤษ)
NotNull=Field {0} is required
NotBlank=Field {0} cannot be blank
Size=Field {0} must be between {2} and {1} characters
Min=Field {0} must be at least {1}
Max=Field {0} must be at most {1}
Email=Field {0} must be a valid email address
Pattern=Field {0} has invalid format

javax.validation.constraints.NotNull.message=This field is required
javax.validation.constraints.NotBlank.message=This field cannot be blank
javax.validation.constraints.Email.message=Please enter a valid email address
```

```properties
# src/main/resources/messages/validation_th.properties (ไทย)
NotNull=กรุณากรอกข้อมูล {0}
NotBlank=กรุณากรอกข้อมูล {0} ห้ามเว้นว่าง
Size={0} ต้องมีความยาวระหว่าง {2} ถึง {1} ตัวอักษร
Min={0} ต้องมีค่าอย่างน้อย {1}
Max={0} ต้องมีค่าไม่เกิน {1}
Email={0} ต้องเป็นรูปแบบอีเมลที่ถูกต้อง
Pattern=รูปแบบของ {0} ไม่ถูกต้อง

javax.validation.constraints.NotNull.message=จำเป็นต้องกรอกข้อมูลนี้
javax.validation.constraints.NotBlank.message=ช่องนี้ห้ามเว้นว่าง
javax.validation.constraints.Email.message=กรุณากรอกอีเมลให้ถูกต้อง
```

---

## 2967-2971: LocaleResolver

LocaleResolver กำหนดว่า Spring จะตัดสินใจ locale ของ request จากที่ไหน

### AcceptHeader LocaleResolver

```java
// LocaleConfig.java
package com.example.i18n.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver;

import java.util.Arrays;
import java.util.List;
import java.util.Locale;

@Configuration
public class LocaleConfig {

    // ใช้ Accept-Language header จาก browser
    @Bean
    public LocaleResolver localeResolver() {
        AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
        
        // กำหนด locales ที่รองรับ
        resolver.setSupportedLocales(Arrays.asList(
            Locale.ENGLISH,
            new Locale("th"),
            Locale.JAPANESE,
            Locale.SIMPLIFIED_CHINESE,
            new Locale("ar")  // Arabic
        ));
        
        // Default locale ถ้าไม่ match
        resolver.setDefaultLocale(Locale.ENGLISH);
        
        return resolver;
    }
}
```

### Cookie LocaleResolver

```java
// CookieLocaleConfig.java - ใช้ cookie เก็บ locale preference ของ user
package com.example.i18n.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.CookieLocaleResolver;

import java.time.Duration;
import java.util.Locale;

@Configuration
public class CookieLocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver resolver = new CookieLocaleResolver("LOCALE");
        resolver.setDefaultLocale(Locale.ENGLISH);
        resolver.setCookieMaxAge(Duration.ofDays(30));
        resolver.setCookieHttpOnly(true);
        resolver.setCookieSecure(true);  // HTTPS only
        return resolver;
    }
}
```

### Session LocaleResolver

```java
// SessionLocaleConfig.java - เก็บ locale ใน session
package com.example.i18n.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;

import java.util.Locale;

@Configuration
public class SessionLocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }
}
```

### Locale Change API

```java
// LocaleController.java
package com.example.i18n.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.LocaleResolver;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

@RestController
@RequestMapping("/api/v1/locale")
@RequiredArgsConstructor
public class LocaleController {

    private final LocaleResolver localeResolver;

    private static final Set<String> SUPPORTED_LOCALES = Set.of(
        "en", "th", "ja", "zh_CN", "ar", "ko", "vi"
    );

    @PostMapping
    public ResponseEntity<Map<String, String>> changeLocale(
        @RequestParam String locale,
        HttpServletRequest request,
        HttpServletResponse response
    ) {
        if (!SUPPORTED_LOCALES.contains(locale)) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "Unsupported locale: " + locale));
        }
        
        Locale newLocale = Locale.forLanguageTag(locale.replace("_", "-"));
        localeResolver.setLocale(request, response, newLocale);
        
        return ResponseEntity.ok(Map.of(
            "locale", newLocale.toString(),
            "language", newLocale.getDisplayLanguage(newLocale),
            "country", newLocale.getCountry()
        ));
    }

    @GetMapping
    public ResponseEntity<Map<String, String>> getCurrentLocale(HttpServletRequest request) {
        Locale locale = localeResolver.resolveLocale(request);
        return ResponseEntity.ok(Map.of(
            "locale", locale.toString(),
            "language", locale.getLanguage(),
            "country", locale.getCountry(),
            "displayLanguage", locale.getDisplayLanguage(locale)
        ));
    }
}
```

---

## 2972-2977: MessageHelper Service

```java
// MessageHelper.java
package com.example.i18n.service;

import lombok.RequiredArgsConstructor;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

import java.util.Locale;

@Component
@RequiredArgsConstructor
public class MessageHelper {

    private final MessageSource messageSource;

    // ดึง message ตาม locale ปัจจุบัน
    public String get(String code, Object... args) {
        return messageSource.getMessage(
            code,
            args,
            LocaleContextHolder.getLocale()
        );
    }

    // ดึง message สำหรับ locale ที่กำหนด
    public String get(String code, Locale locale, Object... args) {
        return messageSource.getMessage(code, args, locale);
    }

    // ดึง message พร้อม default value
    public String getOrDefault(String code, String defaultMessage, Object... args) {
        return messageSource.getMessage(code, args, defaultMessage, 
            LocaleContextHolder.getLocale());
    }

    // ดึง locale ปัจจุบัน
    public Locale getCurrentLocale() {
        return LocaleContextHolder.getLocale();
    }
}
```

### การใช้งานใน Service

```java
// UserService.java
package com.example.i18n.service;

import com.example.i18n.entity.User;
import com.example.i18n.exception.BusinessException;
import com.example.i18n.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final MessageHelper msg;

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new BusinessException(
                msg.get("user.not.found", id)
            ));
    }

    public User createUser(CreateUserRequest request) {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException(
                msg.get("user.email.exists", request.getEmail())
            );
        }
        
        User user = userRepository.save(/* ... */);
        return user;
    }
}
```

---

## 2978-2982: @Valid กับ Localized Error Messages

```java
// CreateUserRequest.java
package com.example.i18n.dto;

import jakarta.validation.constraints.*;
import lombok.*;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CreateUserRequest {

    @NotBlank(message = "{validation.user.name.required}")
    @Size(min = 2, max = 100, message = "{validation.user.name.size}")
    private String name;

    @NotBlank(message = "{validation.user.email.required}")
    @Email(message = "{validation.user.email.invalid}")
    private String email;

    @NotBlank(message = "{validation.user.password.required}")
    @Size(min = 8, max = 50, message = "{validation.user.password.size}")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).*$",
        message = "{validation.user.password.pattern}"
    )
    private String password;

    @NotNull(message = "{validation.user.age.required}")
    @Min(value = 18, message = "{validation.user.age.min}")
    @Max(value = 120, message = "{validation.user.age.max}")
    private Integer age;

    @Pattern(
        regexp = "^\\+?[0-9]{10,15}$",
        message = "{validation.user.phone.pattern}"
    )
    private String phoneNumber;
}
```

```properties
# validation.properties (English)
validation.user.name.required=Name is required
validation.user.name.size=Name must be between 2 and 100 characters
validation.user.email.required=Email is required
validation.user.email.invalid=Please enter a valid email address
validation.user.password.required=Password is required
validation.user.password.size=Password must be between 8 and 50 characters
validation.user.password.pattern=Password must contain at least one uppercase, one lowercase, and one number
validation.user.age.required=Age is required
validation.user.age.min=You must be at least 18 years old
validation.user.age.max=Age cannot exceed 120
validation.user.phone.pattern=Phone number must be 10-15 digits
```

```properties
# validation_th.properties (Thai)
validation.user.name.required=กรุณากรอกชื่อ
validation.user.name.size=ชื่อต้องมีความยาว 2-100 ตัวอักษร
validation.user.email.required=กรุณากรอกอีเมล
validation.user.email.invalid=กรุณากรอกอีเมลให้ถูกต้อง
validation.user.password.required=กรุณากรอกรหัสผ่าน
validation.user.password.size=รหัสผ่านต้องมีความยาว 8-50 ตัวอักษร
validation.user.password.pattern=รหัสผ่านต้องมีตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก และตัวเลขอย่างน้อยอย่างละ 1 ตัว
validation.user.age.required=กรุณากรอกอายุ
validation.user.age.min=คุณต้องมีอายุอย่างน้อย 18 ปี
validation.user.age.max=อายุต้องไม่เกิน 120 ปี
validation.user.phone.pattern=หมายเลขโทรศัพท์ต้องมี 10-15 หลัก
```

### Global Exception Handler

```java
// GlobalExceptionHandler.java
package com.example.i18n.exception;

import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidationErrors(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> fieldErrors = new HashMap<>();
        
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            fieldErrors.put(fieldName, errorMessage);
        });
        
        return ResponseEntity.badRequest().body(Map.of(
            "status", HttpStatus.BAD_REQUEST.value(),
            "error", "Validation Failed",
            "fieldErrors", fieldErrors
        ));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<Map<String, Object>> handleBusinessException(
        BusinessException ex
    ) {
        return ResponseEntity.badRequest().body(Map.of(
            "status", HttpStatus.BAD_REQUEST.value(),
            "error", "Business Error",
            "message", ex.getMessage()
        ));
    }
}
```

---

## 2983-2988: Currency และ Date Formatting

### Number และ Currency Formatter

```java
// LocalizedFormatter.java
package com.example.i18n.util;

import lombok.RequiredArgsConstructor;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.text.NumberFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.FormatStyle;
import java.util.Currency;
import java.util.Locale;
import java.util.Map;

@Component
public class LocalizedFormatter {

    // ตาราง currency ตาม locale
    private static final Map<String, String> LOCALE_CURRENCY = Map.of(
        "th", "THB",
        "ja", "JPY",
        "zh_CN", "CNY",
        "en_US", "USD",
        "en_GB", "GBP",
        "de", "EUR",
        "fr", "EUR",
        "ko", "KRW"
    );

    // Format currency ตาม locale ปัจจุบัน
    public String formatCurrency(BigDecimal amount) {
        return formatCurrency(amount, LocaleContextHolder.getLocale());
    }

    public String formatCurrency(BigDecimal amount, Locale locale) {
        String currencyCode = getCurrencyCode(locale);
        
        NumberFormat formatter = NumberFormat.getCurrencyInstance(locale);
        formatter.setCurrency(Currency.getInstance(currencyCode));
        
        return formatter.format(amount);
    }

    // Format number ตาม locale (จุดทศนิยมต่างกันในแต่ละประเทศ)
    public String formatNumber(BigDecimal number) {
        NumberFormat formatter = NumberFormat.getNumberInstance(
            LocaleContextHolder.getLocale()
        );
        formatter.setMinimumFractionDigits(2);
        formatter.setMaximumFractionDigits(2);
        return formatter.format(number);
    }

    // Format วันที่ตาม locale
    public String formatDate(LocalDate date) {
        return formatDate(date, LocaleContextHolder.getLocale());
    }

    public String formatDate(LocalDate date, Locale locale) {
        DateTimeFormatter formatter = DateTimeFormatter
            .ofLocalizedDate(FormatStyle.MEDIUM)
            .withLocale(locale);
        return date.format(formatter);
    }

    // Format วันที่และเวลาตาม locale
    public String formatDateTime(LocalDateTime dateTime) {
        return formatDateTime(dateTime, LocaleContextHolder.getLocale());
    }

    public String formatDateTime(LocalDateTime dateTime, Locale locale) {
        DateTimeFormatter formatter = DateTimeFormatter
            .ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(locale);
        return dateTime.format(formatter);
    }

    private String getCurrencyCode(Locale locale) {
        String localeKey = locale.getLanguage() + "_" + locale.getCountry();
        String currencyCode = LOCALE_CURRENCY.get(localeKey);
        
        if (currencyCode == null) {
            currencyCode = LOCALE_CURRENCY.getOrDefault(locale.getLanguage(), "USD");
        }
        
        return currencyCode;
    }
}
```

### การใช้งานใน Response

```java
// ProductController.java
package com.example.i18n.controller;

import com.example.i18n.dto.ProductResponse;
import com.example.i18n.service.ProductService;
import com.example.i18n.util.LocalizedFormatter;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Locale;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;
    private final LocalizedFormatter formatter;
    private final MessageHelper msg;

    @GetMapping
    public ResponseEntity<List<ProductResponse>> getProducts(Locale locale) {
        // Locale จะถูก inject อัตโนมัติตาม request
        return ResponseEntity.ok(
            productService.findAll().stream()
                .map(product -> ProductResponse.builder()
                    .id(product.getId())
                    .name(product.getName())
                    .priceFormatted(formatter.formatCurrency(product.getPrice(), locale))
                    .stockStatus(product.getStock() > 0 
                        ? msg.get("product.in.stock", locale)
                        : msg.get("product.out.of.stock", locale, product.getName()))
                    .build())
                .toList()
        );
    }
}
```

---

## 2989-2994: RTL Language Support

ภาษา Arabic และ Hebrew เขียนจากขวาไปซ้าย (RTL - Right-to-Left) ต้องรองรับใน UI

### Locale Response Interceptor

```java
// LocaleInfoInterceptor.java
package com.example.i18n.interceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.LocaleResolver;

import java.util.Locale;
import java.util.Set;

public class LocaleInfoInterceptor implements HandlerInterceptor {

    private static final Set<String> RTL_LANGUAGES = Set.of("ar", "he", "fa", "ur");
    
    private final LocaleResolver localeResolver;

    public LocaleInfoInterceptor(LocaleResolver localeResolver) {
        this.localeResolver = localeResolver;
    }

    @Override
    public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) {
        Locale locale = localeResolver.resolveLocale(request);
        
        // เพิ่ม headers เกี่ยวกับ locale
        response.setHeader("X-Locale", locale.toString());
        response.setHeader("X-Language", locale.getLanguage());
        response.setHeader("X-Text-Direction", 
            RTL_LANGUAGES.contains(locale.getLanguage()) ? "rtl" : "ltr");
        
        return true;
    }
}
```

### Locale Info Response Wrapper

```java
// LocalizedResponse.java
package com.example.i18n.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.*;

import java.util.Locale;
import java.util.Set;

@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class LocalizedResponse<T> {
    private T data;
    private LocaleInfo locale;
    
    @Data
    @Builder
    public static class LocaleInfo {
        private String code;
        private String language;
        private String country;
        private String direction;
        private String displayName;
    }
    
    private static final Set<String> RTL_LANGUAGES = Set.of("ar", "he", "fa", "ur");
    
    public static <T> LocalizedResponse<T> of(T data, Locale locale) {
        String direction = RTL_LANGUAGES.contains(locale.getLanguage()) ? "rtl" : "ltr";
        
        return LocalizedResponse.<T>builder()
            .data(data)
            .locale(LocaleInfo.builder()
                .code(locale.toString())
                .language(locale.getLanguage())
                .country(locale.getCountry())
                .direction(direction)
                .displayName(locale.getDisplayLanguage(locale))
                .build())
            .build();
    }
}
```

---

## 2995-3000: Content Negotiation สำหรับ Language

### Language-aware Content Negotiation

```java
// LanguageContentNegotiationConfig.java
package com.example.i18n.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ContentNegotiationConfigurer;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class LanguageContentNegotiationConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)           // รองรับ ?format=json
            .parameterName("format")
            .favorPathExtension(false)
            .ignoreAcceptHeader(false)      // ใช้ Accept header
            .defaultContentType(
                org.springframework.http.MediaType.APPLICATION_JSON
            );
    }
}
```

### Multilingual Response DTO

```java
// MultilingualProductResponse.java
package com.example.i18n.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.*;

import java.math.BigDecimal;
import java.util.Map;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class MultilingualProductResponse {
    private Long id;
    
    // ชื่อในหลายภาษา
    private Map<String, String> names;
    
    // คำอธิบายในหลายภาษา
    private Map<String, String> descriptions;
    
    // ราคา formatted ตาม locale
    private String priceFormatted;
    
    private BigDecimal price;
    private String currencyCode;
    private Integer stock;
}
```

### Multilingual Entity

```java
// ProductTranslation.java
package com.example.i18n.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "product_translations",
    uniqueConstraints = @UniqueConstraint(columnNames = {"product_id", "locale"}))
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ProductTranslation {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false, length = 10)
    private String locale;  // "en", "th", "ja"

    @Column(nullable = false, length = 200)
    private String name;

    @Column(length = 2000)
    private String description;

    @Column(length = 500)
    private String shortDescription;
}
```

### Locale-aware Repository

```java
// ProductRepository.java
package com.example.i18n.repository;

import com.example.i18n.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("""
        SELECT p FROM Product p
        LEFT JOIN FETCH p.translations t
        WHERE t.locale = :locale OR t.locale = 'en'
        ORDER BY p.id
        """)
    List<Product> findAllWithTranslation(@Param("locale") String locale);

    @Query("""
        SELECT p FROM Product p
        LEFT JOIN FETCH p.translations t
        WHERE p.id = :id AND (t.locale = :locale OR t.locale = 'en')
        """)
    Optional<Product> findByIdWithTranslation(
        @Param("id") Long id,
        @Param("locale") String locale
    );
}
```

### Locale Test

```java
// LocaleTest.java
package com.example.i18n;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class LocaleTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testEnglishLocale() throws Exception {
        mockMvc.perform(get("/api/v1/products")
            .header("Accept-Language", "en"))
            .andExpect(status().isOk())
            .andExpect(header().string("X-Language", "en"))
            .andExpect(header().string("X-Text-Direction", "ltr"));
    }

    @Test
    void testThaiLocale() throws Exception {
        mockMvc.perform(get("/api/v1/products")
            .header("Accept-Language", "th"))
            .andExpect(status().isOk())
            .andExpect(header().string("X-Language", "th"))
            .andExpect(header().string("X-Text-Direction", "ltr"));
    }

    @Test
    void testArabicRTL() throws Exception {
        mockMvc.perform(get("/api/v1/products")
            .header("Accept-Language", "ar"))
            .andExpect(status().isOk())
            .andExpect(header().string("X-Language", "ar"))
            .andExpect(header().string("X-Text-Direction", "rtl"));
    }

    @Test
    void testValidationMessagesInThai() throws Exception {
        mockMvc.perform(
            org.springframework.test.web.servlet.request.MockMvcRequestBuilders
                .post("/api/v1/users")
                .contentType("application/json")
                .header("Accept-Language", "th")
                .content("{\"name\":\"\",\"email\":\"invalid\",\"age\":15}")
        )
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.fieldErrors.name").value("กรุณากรอกชื่อ"))
        .andExpect(jsonPath("$.fieldErrors.email").value("กรุณากรอกอีเมลให้ถูกต้อง"));
    }
}
```

---

## สรุป Part 84

ในบทนี้เราได้เรียนรู้:

1. **MessageSource** - การตั้งค่าและใช้งาน message bundles
2. **LocaleResolver** - AcceptHeader, Cookie, Session resolver
3. **messages.properties** - การสร้าง message files หลายภาษา
4. **@Valid + i18n** - Validation messages ในแต่ละภาษา
5. **Currency & Date** - การ format ตาม locale
6. **RTL Support** - การรองรับภาษาที่เขียนจากขวาไปซ้าย
7. **Content Negotiation** - Language-aware content negotiation

---

*[← Part 83: Audit Trail](./part-83-audit-trail.md) | [Part 85: Health Monitoring →](./part-85-health-monitoring.md)*

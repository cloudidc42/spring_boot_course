# Part 14: Validation และ Error Handling
## ขั้นตอนที่ 331-360

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** Validation และ Error Handling ระดับ production

---

## ขั้นตอนที่ 331: Bean Validation Annotations

```java
public record CreateUserRequest(
    
    // String validations
    @NotNull(message = "Username is required")
    @NotBlank(message = "Username cannot be blank")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username can only contain letters, numbers, underscore")
    String username,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email format is invalid")
    @Size(max = 255)
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    String password,
    
    // Number validations
    @Min(value = 0, message = "Age cannot be negative")
    @Max(value = 150, message = "Age is too large")
    Integer age,
    
    @DecimalMin(value = "0.0", inclusive = false, message = "Price must be greater than 0")
    @DecimalMax(value = "999999.99", message = "Price is too large")
    @Digits(integer = 6, fraction = 2, message = "Invalid price format")
    BigDecimal price,
    
    // Date validations
    @Past(message = "Birth date must be in the past")
    LocalDate birthDate,
    
    @Future(message = "Expiry date must be in the future")
    LocalDate expiryDate,
    
    @FutureOrPresent(message = "Start date must be now or future")
    LocalDateTime startAt,
    
    // Collection validations
    @NotEmpty(message = "Tags cannot be empty")
    @Size(max = 10, message = "Maximum 10 tags")
    List<@NotBlank String> tags,
    
    // Nested object validation
    @Valid
    @NotNull
    AddressRequest address,
    
    // Boolean
    @AssertTrue(message = "Must agree to terms")
    Boolean agreeToTerms
) {}
```

---

## ขั้นตอนที่ 332: Custom Validator - Field Level

```java
// 1. Annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ThaiPhoneValidator.class)
@Documented
public @interface ThaiPhone {
    String message() default "Invalid Thai phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. Validator
public class ThaiPhoneValidator implements ConstraintValidator<ThaiPhone, String> {
    
    private static final Pattern THAI_PHONE = Pattern.compile("^(0[689]\\d{8}|\\+66[689]\\d{8})$");
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isBlank()) {
            return true;  // null check = @NotNull
        }
        return THAI_PHONE.matcher(value).matches();
    }
}

// 3. Use
public record CreateUserRequest(
    @ThaiPhone
    String phone
) {}
```

---

## ขั้นตอนที่ 333: Custom Validator - Class Level (Cross-field)

```java
// Annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String passwordField() default "password";
    String confirmField() default "confirmPassword";
}

// Validator
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, Object> {
    
    private String passwordField;
    private String confirmField;
    
    @Override
    public void initialize(PasswordMatch constraintAnnotation) {
        this.passwordField = constraintAnnotation.passwordField();
        this.confirmField = constraintAnnotation.confirmField();
    }
    
    @Override
    public boolean isValid(Object obj, ConstraintValidatorContext context) {
        try {
            Object password = BeanWrapper.getProperty(obj, passwordField);
            Object confirm = BeanWrapper.getProperty(obj, confirmField);
            
            boolean isValid = password != null && password.equals(confirm);
            
            if (!isValid) {
                // Custom error location
                context.disableDefaultConstraintViolation();
                context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                    .addPropertyNode(confirmField)
                    .addConstraintViolation();
            }
            
            return isValid;
        } catch (Exception e) {
            return false;
        }
    }
}

// Use
@PasswordMatch
public record RegisterRequest(
    String username,
    String email,
    String password,
    String confirmPassword
) {}
```

---

## ขั้นตอนที่ 334: Validation Groups

```java
// Groups
public interface OnCreate {}
public interface OnUpdate {}

// Request
public record ProductRequest(
    
    @NotBlank(groups = OnCreate.class)
    @Size(max = 255)
    String name,
    
    @NotNull(groups = OnCreate.class)
    @DecimalMin("0.01")
    BigDecimal price,
    
    @NotNull(groups = OnCreate.class)
    @Min(0)
    Integer stock
) {}

// Controller with @Validated
@PostMapping
public ResponseEntity<?> create(
    @Validated(OnCreate.class) @RequestBody ProductRequest request
) { ... }

@PutMapping("/{id}")
public ResponseEntity<?> update(
    @PathVariable Long id,
    @Validated(OnUpdate.class) @RequestBody ProductRequest request
) { ... }
```

---

## ขั้นตอนที่ 335: Programmatic Validation

```java
@Service
@RequiredArgsConstructor
public class ProductService {
    
    private final javax.validation.Validator validator;
    
    public ProductResponse create(CreateProductRequest request) {
        // Manual validation
        Set<ConstraintViolation<CreateProductRequest>> violations = validator.validate(request);
        
        if (!violations.isEmpty()) {
            Map<String, String> errors = violations.stream()
                .collect(Collectors.toMap(
                    v -> v.getPropertyPath().toString(),
                    ConstraintViolation::getMessage
                ));
            throw new ValidationException(errors);
        }
        
        // Continue...
    }
}
```

---

## ขั้นตอนที่ 336: Custom Exception Hierarchy

```java
// Base exception
public abstract class AppException extends RuntimeException {
    
    private final HttpStatus status;
    private final String errorCode;
    
    public AppException(String message, HttpStatus status, String errorCode) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }
    
    public HttpStatus getStatus() { return status; }
    public String getErrorCode() { return errorCode; }
}

// Specific exceptions
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, String field, Object value) {
        super(
            String.format("%s not found with %s: %s", resource, field, value),
            HttpStatus.NOT_FOUND,
            "RESOURCE_NOT_FOUND"
        );
    }
}

public class DuplicateResourceException extends AppException {
    public DuplicateResourceException(String resource, String field, Object value) {
        super(
            String.format("%s already exists with %s: %s", resource, field, value),
            HttpStatus.CONFLICT,
            "DUPLICATE_RESOURCE"
        );
    }
}

public class InsufficientStockException extends AppException {
    public InsufficientStockException(String product, int requested) {
        super(
            String.format("Insufficient stock for '%s'. Requested: %d", product, requested),
            HttpStatus.BAD_REQUEST,
            "INSUFFICIENT_STOCK"
        );
    }
}

public class BusinessException extends AppException {
    public BusinessException(String message) {
        super(message, HttpStatus.UNPROCESSABLE_ENTITY, "BUSINESS_ERROR");
    }
}

public class UnauthorizedException extends AppException {
    public UnauthorizedException(String message) {
        super(message, HttpStatus.UNAUTHORIZED, "UNAUTHORIZED");
    }
}

public class ForbiddenException extends AppException {
    public ForbiddenException(String message) {
        super(message, HttpStatus.FORBIDDEN, "FORBIDDEN");
    }
}
```

---

## ขั้นตอนที่ 337: Error Response Format

```java
@Getter @Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    
    private final boolean success = false;
    private final String errorCode;
    private final String message;
    private final Map<String, String> fieldErrors;
    private final String path;
    private final String timestamp;
    private final String traceId;
    
    public static ErrorResponse of(AppException ex, String path) {
        return ErrorResponse.builder()
            .errorCode(ex.getErrorCode())
            .message(ex.getMessage())
            .path(path)
            .timestamp(LocalDateTime.now().toString())
            .traceId(MDC.get("requestId"))
            .build();
    }
    
    public static ErrorResponse ofValidation(MethodArgumentNotValidException ex, String path) {
        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> Optional.ofNullable(fe.getDefaultMessage()).orElse("Invalid"),
                (a, b) -> a
            ));
        
        return ErrorResponse.builder()
            .errorCode("VALIDATION_ERROR")
            .message("Validation failed")
            .fieldErrors(fieldErrors)
            .path(path)
            .timestamp(LocalDateTime.now().toString())
            .build();
    }
}
```

---

## ขั้นตอนที่ 338: Complete Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    // AppException subclasses
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(
        AppException ex, HttpServletRequest request
    ) {
        log.warn("App exception: {} - {}", ex.getErrorCode(), ex.getMessage());
        return ResponseEntity
            .status(ex.getStatus())
            .body(ErrorResponse.of(ex, request.getRequestURI()));
    }
    
    // Validation
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(
        MethodArgumentNotValidException ex, HttpServletRequest request
    ) {
        return ErrorResponse.ofValidation(ex, request.getRequestURI());
    }
    
    // @PathVariable type mismatch
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleTypeMismatch(MethodArgumentTypeMismatchException ex) {
        return ErrorResponse.builder()
            .errorCode("TYPE_MISMATCH")
            .message(String.format("Invalid value '%s' for parameter '%s'", 
                ex.getValue(), ex.getName()))
            .build();
    }
    
    // Missing required parameter
    @ExceptionHandler(MissingServletRequestParameterException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleMissingParam(MissingServletRequestParameterException ex) {
        return ErrorResponse.builder()
            .errorCode("MISSING_PARAMETER")
            .message("Required parameter missing: " + ex.getParameterName())
            .build();
    }
    
    // HTTP method not supported
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    public ErrorResponse handleMethodNotAllowed(HttpRequestMethodNotSupportedException ex) {
        return ErrorResponse.builder()
            .errorCode("METHOD_NOT_ALLOWED")
            .message(ex.getMessage())
            .build();
    }
    
    // Media type not supported  
    @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
    @ResponseStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
    public ErrorResponse handleMediaType(HttpMediaTypeNotSupportedException ex) {
        return ErrorResponse.builder()
            .errorCode("UNSUPPORTED_MEDIA_TYPE")
            .message("Unsupported content type: " + ex.getContentType())
            .build();
    }
    
    // JSON parse error
    @ExceptionHandler(HttpMessageNotReadableException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleJsonError(HttpMessageNotReadableException ex) {
        return ErrorResponse.builder()
            .errorCode("INVALID_JSON")
            .message("Invalid request body")
            .build();
    }
    
    // DataIntegrityViolation (DB constraint)
    @ExceptionHandler(DataIntegrityViolationException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleDataIntegrity(DataIntegrityViolationException ex) {
        log.warn("Data integrity violation: {}", ex.getMessage());
        
        String message = "Data integrity violation";
        if (ex.getMessage() != null && ex.getMessage().contains("unique constraint")) {
            message = "Duplicate value violates unique constraint";
        }
        
        return ErrorResponse.builder()
            .errorCode("DATA_INTEGRITY_ERROR")
            .message(message)
            .build();
    }
    
    // Fallback
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception at {}: {}", request.getRequestURI(), ex.getMessage(), ex);
        
        return ErrorResponse.builder()
            .errorCode("INTERNAL_SERVER_ERROR")
            .message("An unexpected error occurred")
            .traceId(MDC.get("requestId"))
            .build();
    }
}
```

---

## ขั้นตอนที่ 339: Constraint Violation Messages (i18n)

```properties
# src/main/resources/messages.properties (default)
jakarta.validation.constraints.NotNull.message=This field is required
jakarta.validation.constraints.NotBlank.message=This field cannot be blank
jakarta.validation.constraints.Size.message=Size must be between {min} and {max}
jakarta.validation.constraints.Email.message=Invalid email format
jakarta.validation.constraints.Min.message=Value must be at least {value}
jakarta.validation.constraints.Max.message=Value must be at most {value}

# Custom validators
validator.phone.thai=Invalid Thai phone number (format: 08x-xxxx-xxxx)
validator.password.match=Passwords do not match
```

```java
// Use message key
public @interface ThaiPhone {
    String message() default "{validator.phone.thai}";
}
```

---

## ขั้นตอนที่ 340-360: Request/Response Logging

```java
// Log all requests and responses
@Component
@Slf4j
public class RequestResponseLoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request, 
        HttpServletResponse response, 
        FilterChain chain
    ) throws ServletException, IOException {
        
        ContentCachingRequestWrapper wrappedRequest = 
            new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper wrappedResponse = 
            new ContentCachingResponseWrapper(response);
        
        long start = System.currentTimeMillis();
        
        try {
            chain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            long duration = System.currentTimeMillis() - start;
            
            String requestBody = getBody(wrappedRequest.getContentAsByteArray());
            String responseBody = getBody(wrappedResponse.getContentAsByteArray());
            
            if (log.isDebugEnabled()) {
                log.debug("""
                    HTTP {} {} -> {} ({}ms)
                    Request: {}
                    Response: {}
                    """,
                    request.getMethod(),
                    request.getRequestURI(),
                    response.getStatus(),
                    duration,
                    truncate(requestBody, 500),
                    truncate(responseBody, 500)
                );
            }
            
            wrappedResponse.copyBodyToResponse();
        }
    }
    
    private String getBody(byte[] bytes) {
        if (bytes == null || bytes.length == 0) return "";
        return new String(bytes, StandardCharsets.UTF_8);
    }
    
    private String truncate(String s, int max) {
        if (s == null || s.length() <= max) return s;
        return s.substring(0, max) + "... [truncated]";
    }
}
```

---

*[← Part 13: CRUD Operations](./part-13-crud-operations.md) | [Part 15: Relationships →](./part-15-entity-relationships.md)*

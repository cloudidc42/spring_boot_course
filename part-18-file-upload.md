# Part 18: File Upload และ Storage
## ขั้นตอนที่ 461-490

> **ระดับ:** กลาง (Intermediate)  
> **เวลาเรียน:** 4-5 ชั่วโมง  
> **เป้าหมาย:** File upload ที่ปลอดภัยและ scalable

---

## ขั้นตอนที่ 461: File Upload Configuration

```yaml
# application.yml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 10MB
      max-request-size: 50MB
      file-size-threshold: 2KB

app:
  upload:
    directory: ${UPLOAD_DIR:./uploads}
    allowed-types: image/jpeg,image/png,image/gif,image/webp,application/pdf
    max-file-size: 10485760  # 10MB
    image-max-width: 2000
    image-max-height: 2000
```

```java
@Configuration
@ConfigurationProperties(prefix = "app.upload")
@Getter @Setter
public class UploadConfig {
    private String directory = "./uploads";
    private Set<String> allowedTypes = Set.of("image/jpeg", "image/png", "image/gif");
    private long maxFileSize = 10 * 1024 * 1024;  // 10MB
    private int imageMaxWidth = 2000;
    private int imageMaxHeight = 2000;
}
```

---

## ขั้นตอนที่ 462: File Storage Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class LocalFileStorageService implements FileStorageService {
    
    private final UploadConfig uploadConfig;
    private Path rootLocation;
    
    @PostConstruct
    public void init() {
        rootLocation = Paths.get(uploadConfig.getDirectory());
        try {
            Files.createDirectories(rootLocation);
            log.info("Upload directory: {}", rootLocation.toAbsolutePath());
        } catch (IOException e) {
            throw new StorageException("Could not initialize upload directory", e);
        }
    }
    
    @Override
    public String store(MultipartFile file, String folder) {
        validateFile(file);
        
        String filename = generateFilename(file.getOriginalFilename());
        Path targetDir = rootLocation.resolve(folder);
        
        try {
            Files.createDirectories(targetDir);
            Path target = targetDir.resolve(filename);
            
            // Prevent path traversal attack
            if (!target.normalize().startsWith(rootLocation.normalize())) {
                throw new StorageException("Cannot store file outside upload directory");
            }
            
            Files.copy(file.getInputStream(), target, StandardCopyOption.REPLACE_EXISTING);
            
            log.info("File stored: {}/{}", folder, filename);
            return folder + "/" + filename;
            
        } catch (IOException e) {
            throw new StorageException("Failed to store file: " + filename, e);
        }
    }
    
    @Override
    public Resource load(String path) {
        try {
            Path file = rootLocation.resolve(path).normalize();
            
            // Security: prevent directory traversal
            if (!file.startsWith(rootLocation)) {
                throw new StorageException("Access denied: " + path);
            }
            
            Resource resource = new UrlResource(file.toUri());
            if (resource.exists() && resource.isReadable()) {
                return resource;
            }
            throw new ResourceNotFoundException("File", "path", path);
        } catch (MalformedURLException e) {
            throw new StorageException("Could not read file: " + path, e);
        }
    }
    
    @Override
    public void delete(String path) {
        try {
            Path file = rootLocation.resolve(path).normalize();
            if (!file.startsWith(rootLocation)) {
                throw new StorageException("Access denied");
            }
            Files.deleteIfExists(file);
        } catch (IOException e) {
            throw new StorageException("Failed to delete file: " + path, e);
        }
    }
    
    private void validateFile(MultipartFile file) {
        if (file.isEmpty()) {
            throw new InvalidFileException("File is empty");
        }
        
        if (file.getSize() > uploadConfig.getMaxFileSize()) {
            throw new InvalidFileException(String.format(
                "File size %d exceeds maximum %d bytes",
                file.getSize(), uploadConfig.getMaxFileSize()
            ));
        }
        
        String contentType = file.getContentType();
        if (contentType == null || !uploadConfig.getAllowedTypes().contains(contentType)) {
            throw new InvalidFileException("File type not allowed: " + contentType);
        }
        
        // Check actual content type (not just header)
        validateMimeType(file);
    }
    
    private void validateMimeType(MultipartFile file) {
        try {
            Tika tika = new Tika();
            String detectedType = tika.detect(file.getInputStream());
            
            if (!uploadConfig.getAllowedTypes().contains(detectedType)) {
                throw new InvalidFileException("Invalid file content: " + detectedType);
            }
        } catch (IOException e) {
            throw new InvalidFileException("Cannot read file");
        }
    }
    
    private String generateFilename(String originalFilename) {
        String extension = StringUtils.getFilenameExtension(originalFilename);
        return UUID.randomUUID().toString() + (extension != null ? "." + extension : "");
    }
}
```

---

## ขั้นตอนที่ 463: File Upload Controller

```java
@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
@Tag(name = "Files")
@PreAuthorize("isAuthenticated()")
public class FileController {
    
    private final FileStorageService storageService;
    
    @PostMapping("/upload")
    public ResponseEntity<ApiResponse<FileResponse>> upload(
        @RequestParam("file") MultipartFile file,
        @RequestParam(defaultValue = "general") String folder
    ) {
        String path = storageService.store(file, folder);
        String url = "/api/v1/files/" + path;
        
        FileResponse response = new FileResponse(
            path,
            url,
            file.getOriginalFilename(),
            file.getContentType(),
            file.getSize()
        );
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(response, "File uploaded successfully"));
    }
    
    @PostMapping("/upload/multiple")
    public ResponseEntity<ApiResponse<List<FileResponse>>> uploadMultiple(
        @RequestParam("files") List<MultipartFile> files,
        @RequestParam(defaultValue = "general") String folder
    ) {
        if (files.size() > 10) {
            throw new InvalidFileException("Maximum 10 files per request");
        }
        
        List<FileResponse> responses = files.stream()
            .map(file -> {
                String path = storageService.store(file, folder);
                return new FileResponse(path, "/api/v1/files/" + path,
                    file.getOriginalFilename(), file.getContentType(), file.getSize());
            })
            .toList();
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(responses));
    }
    
    @GetMapping("/**")
    @PreAuthorize("permitAll()")
    public ResponseEntity<Resource> download(HttpServletRequest request) {
        String path = request.getRequestURI().replaceFirst("/api/v1/files/", "");
        
        Resource resource = storageService.load(path);
        
        String contentType = null;
        try {
            contentType = request.getServletContext().getMimeType(resource.getFile().getAbsolutePath());
        } catch (IOException e) {
            contentType = MediaType.APPLICATION_OCTET_STREAM_VALUE;
        }
        
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType != null ? contentType : "application/octet-stream"))
            .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"" + resource.getFilename() + "\"")
            .body(resource);
    }
    
    @DeleteMapping("/**")
    public ResponseEntity<ApiResponse<Void>> delete(HttpServletRequest request) {
        String path = request.getRequestURI().replaceFirst("/api/v1/files/", "");
        storageService.delete(path);
        return ResponseEntity.ok(ApiResponse.success(null, "File deleted"));
    }
}
```

---

## ขั้นตอนที่ 464: Image Processing

```java
// Add dependency: net.coobird:thumbnailator
@Service
@RequiredArgsConstructor
@Slf4j
public class ImageProcessingService {
    
    private final UploadConfig config;
    
    public byte[] resize(MultipartFile file, int width, int height) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();
        
        Thumbnails.of(file.getInputStream())
            .size(width, height)
            .keepAspectRatio(true)
            .outputFormat(getFormat(file.getContentType()))
            .toOutputStream(output);
        
        return output.toByteArray();
    }
    
    public Map<String, byte[]> generateThumbnails(MultipartFile file) throws IOException {
        Map<String, byte[]> thumbnails = new HashMap<>();
        
        int[][] sizes = {{150, 150}, {300, 300}, {600, 600}};
        String[] names = {"thumbnail", "medium", "large"};
        
        for (int i = 0; i < sizes.length; i++) {
            thumbnails.put(names[i], resize(file, sizes[i][0], sizes[i][1]));
        }
        
        return thumbnails;
    }
    
    private String getFormat(String contentType) {
        return switch (contentType) {
            case "image/jpeg" -> "jpg";
            case "image/png" -> "png";
            case "image/gif" -> "gif";
            case "image/webp" -> "webp";
            default -> "jpg";
        };
    }
}
```

---

## ขั้นตอนที่ 465: AWS S3 Storage

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.21.0</version>
</dependency>
```

```java
@Service
@Profile("cloud")  // Use only in cloud profile
@RequiredArgsConstructor
@Slf4j
public class S3FileStorageService implements FileStorageService {
    
    private final S3Client s3Client;
    
    @Value("${aws.s3.bucket}")
    private String bucket;
    
    @Value("${aws.s3.region}")
    private String region;
    
    @Override
    public String store(MultipartFile file, String folder) {
        validateFile(file);
        
        String key = folder + "/" + UUID.randomUUID() + getExtension(file.getOriginalFilename());
        
        try {
            PutObjectRequest putRequest = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(file.getContentType())
                .contentLength(file.getSize())
                .build();
            
            s3Client.putObject(putRequest, RequestBody.fromInputStream(
                file.getInputStream(), file.getSize()));
            
            log.info("File uploaded to S3: {}", key);
            return key;
            
        } catch (IOException | S3Exception e) {
            throw new StorageException("Failed to upload to S3: " + e.getMessage(), e);
        }
    }
    
    public String getPresignedUrl(String key, Duration expiry) {
        S3Presigner presigner = S3Presigner.builder().region(Region.of(region)).build();
        
        GetObjectPresignRequest request = GetObjectPresignRequest.builder()
            .signatureDuration(expiry)
            .getObjectRequest(r -> r.bucket(bucket).key(key))
            .build();
        
        return presigner.presignGetObject(request).url().toString();
    }
    
    @Override
    public Resource load(String key) {
        try {
            GetObjectRequest getRequest = GetObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build();
            
            ResponseBytes<GetObjectResponse> bytes = s3Client.getObjectAsBytes(getRequest);
            return new ByteArrayResource(bytes.asByteArray());
        } catch (S3Exception e) {
            throw new ResourceNotFoundException("File", "key", key);
        }
    }
    
    @Override
    public void delete(String key) {
        s3Client.deleteObject(DeleteObjectRequest.builder().bucket(bucket).key(key).build());
    }
    
    private String getExtension(String filename) {
        if (filename == null) return "";
        int dot = filename.lastIndexOf(".");
        return dot >= 0 ? filename.substring(dot) : "";
    }
}
```

---

## ขั้นตอนที่ 466: Profile Image Upload

```java
// User Profile - upload avatar
@RestController
@RequestMapping("/api/v1/users/{userId}/avatar")
@RequiredArgsConstructor
@PreAuthorize("isAuthenticated()")
public class UserAvatarController {
    
    private final UserService userService;
    private final FileStorageService storageService;
    private final ImageProcessingService imageService;
    
    @PostMapping
    public ResponseEntity<ApiResponse<UserResponse>> uploadAvatar(
        @PathVariable Long userId,
        @RequestParam("file") MultipartFile file,
        @AuthenticationPrincipal User currentUser
    ) {
        // Authorization: can only update own avatar (or admin)
        if (!currentUser.getId().equals(userId) && !currentUser.getRole().equals(UserRole.ADMIN)) {
            throw new ForbiddenException("Cannot update another user's avatar");
        }
        
        // Validate image
        if (!file.getContentType().startsWith("image/")) {
            throw new InvalidFileException("Only image files are allowed");
        }
        
        // Resize to standard size
        byte[] resized;
        try {
            resized = imageService.resize(file, 200, 200);
        } catch (IOException e) {
            throw new StorageException("Failed to process image", e);
        }
        
        // Store
        MultipartFile resizedFile = new MockMultipartFile(
            file.getName(),
            file.getOriginalFilename(),
            file.getContentType(),
            resized
        );
        
        String path = storageService.store(resizedFile, "avatars");
        
        // Update user profile
        UserResponse user = userService.updateAvatar(userId, "/api/v1/files/" + path);
        
        return ResponseEntity.ok(ApiResponse.success(user, "Avatar updated"));
    }
    
    @DeleteMapping
    public ResponseEntity<ApiResponse<UserResponse>> deleteAvatar(@PathVariable Long userId) {
        UserResponse user = userService.removeAvatar(userId);
        return ResponseEntity.ok(ApiResponse.success(user, "Avatar removed"));
    }
}
```

---

## ขั้นตอนที่ 467-490: File Upload DTOs & Exceptions

```java
// FileResponse DTO
public record FileResponse(
    String path,
    String url,
    String originalFilename,
    String contentType,
    long size
) {}

// FileStorageService Interface
public interface FileStorageService {
    String store(MultipartFile file, String folder);
    Resource load(String path);
    void delete(String path);
}

// Custom Exceptions
public class StorageException extends AppException {
    public StorageException(String message) {
        super(message, HttpStatus.INTERNAL_SERVER_ERROR, "STORAGE_ERROR");
    }
    public StorageException(String message, Throwable cause) {
        super(message, HttpStatus.INTERNAL_SERVER_ERROR, "STORAGE_ERROR");
        initCause(cause);
    }
}

public class InvalidFileException extends AppException {
    public InvalidFileException(String message) {
        super(message, HttpStatus.BAD_REQUEST, "INVALID_FILE");
    }
}

// Upload Security Checklist
// ✅ Validate MIME type from actual content (Tika), not header
// ✅ Limit file size
// ✅ Limit file types
// ✅ Generate random filenames
// ✅ Prevent directory traversal (normalize path)
// ✅ Store outside web root (in production)
// ✅ Virus scan (ClamAV integration in enterprise)
// ✅ Authentication required
// ✅ Authorization: users can only manage their own files
// ✅ Rate limit upload endpoints
```

---

*[← Part 17: JWT Advanced](./part-17-jwt-advanced.md) | [Part 19: Email Service →](./part-19-email-service.md)*

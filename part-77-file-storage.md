# Part 77: File Storage
## ขั้นตอนที่ 2681-2720

**ระดับ:** ระดับสูง (Advanced)
**เวลาเรียน:** 4-5 ชั่วโมง
**เป้าหมาย:** เรียนรู้การจัดการไฟล์ใน Spring Boot ตั้งแต่การ upload พื้นฐาน ไปจนถึง AWS S3, Presigned URLs, image resizing และ CDN integration

---

## ขั้นตอนที่ 2681: File Upload พื้นฐาน

Spring Boot รองรับการ upload ไฟล์ผ่าน Multipart ซึ่งต้องกำหนด configuration ก่อน

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-s3</artifactId>
        <version>3.1.0</version>
    </dependency>
    <dependency>
        <groupId>net.coobird</groupId>
        <artifactId>thumbnailator</artifactId>
        <version>0.4.20</version>
    </dependency>
    <dependency>
        <groupId>org.apache.tika</groupId>
        <artifactId>tika-core</artifactId>
        <version>2.9.1</version>
    </dependency>
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-java-sdk-s3</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 50MB
      max-request-size: 100MB
      file-size-threshold: 2MB  # เก็บใน memory ถ้าเล็กกว่านี้

  cloud:
    aws:
      credentials:
        access-key: ${AWS_ACCESS_KEY}
        secret-key: ${AWS_SECRET_KEY}
      region:
        static: ap-southeast-1
      s3:
        bucket: my-app-bucket

app:
  storage:
    allowed-types: image/jpeg,image/png,image/gif,image/webp,application/pdf
    max-size-bytes: 52428800  # 50MB
    upload-dir: /tmp/uploads
    cdn-url: https://cdn.example.com
```

---

## ขั้นตอนที่ 2682: File Validation Service

การตรวจสอบไฟล์ที่ upload เข้ามาเป็นสิ่งสำคัญมากด้านความปลอดภัย

```java
// service/FileValidationService.java
package com.example.storage.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.tika.Tika;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class FileValidationService {

    private final Tika tika = new Tika();

    @Value("${app.storage.allowed-types}")
    private String[] allowedTypes;

    @Value("${app.storage.max-size-bytes}")
    private long maxSizeBytes;

    // ชื่อ extension อันตรายที่ไม่ควรอนุญาต
    private static final List<String> DANGEROUS_EXTENSIONS = Arrays.asList(
        "exe", "bat", "cmd", "sh", "php", "js", "html",
        "jar", "war", "py", "rb", "pl"
    );

    public ValidationResult validate(MultipartFile file) {
        // ตรวจสอบว่าไฟล์ไม่ว่างเปล่า
        if (file == null || file.isEmpty()) {
            return ValidationResult.fail("ไม่มีไฟล์ที่ต้องการ upload");
        }

        // ตรวจสอบขนาดไฟล์
        if (file.getSize() > maxSizeBytes) {
            return ValidationResult.fail(
                String.format("ขนาดไฟล์เกินกำหนด (สูงสุด %d MB)", maxSizeBytes / 1024 / 1024)
            );
        }

        // ตรวจสอบชื่อไฟล์
        String originalFilename = file.getOriginalFilename();
        if (originalFilename == null || originalFilename.contains("..") || originalFilename.contains("/")) {
            return ValidationResult.fail("ชื่อไฟล์ไม่ถูกต้อง");
        }

        // ตรวจสอบ extension
        String extension = getExtension(originalFilename).toLowerCase();
        if (DANGEROUS_EXTENSIONS.contains(extension)) {
            return ValidationResult.fail("ประเภทไฟล์นี้ไม่ได้รับอนุญาต");
        }

        // ตรวจสอบ MIME type จากเนื้อหาไฟล์จริง (ไม่ใช่ header)
        String detectedMimeType;
        try {
            detectedMimeType = tika.detect(file.getBytes());
        } catch (IOException e) {
            return ValidationResult.fail("ไม่สามารถตรวจสอบประเภทไฟล์ได้");
        }

        boolean typeAllowed = Arrays.stream(allowedTypes)
            .anyMatch(allowed -> allowed.equals(detectedMimeType));

        if (!typeAllowed) {
            log.warn("Rejected file with detected type: {}", detectedMimeType);
            return ValidationResult.fail(
                "ประเภทไฟล์ไม่ได้รับอนุญาต (detected: " + detectedMimeType + ")"
            );
        }

        return ValidationResult.ok(detectedMimeType);
    }

    private String getExtension(String filename) {
        int lastDot = filename.lastIndexOf('.');
        if (lastDot < 0 || lastDot == filename.length() - 1) return "";
        return filename.substring(lastDot + 1);
    }

    public record ValidationResult(boolean valid, String message, String mimeType) {
        static ValidationResult ok(String mimeType) {
            return new ValidationResult(true, null, mimeType);
        }
        static ValidationResult fail(String message) {
            return new ValidationResult(false, message, null);
        }
    }
}
```

---

## ขั้นตอนที่ 2683: AWS S3 Service

```java
// service/S3StorageService.java
package com.example.storage.service;

import io.awspring.cloud.s3.ObjectMetadata;
import io.awspring.cloud.s3.S3Operations;
import io.awspring.cloud.s3.S3Resource;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.*;

import java.io.InputStream;
import java.net.URL;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class S3StorageService {

    private final S3Operations s3Operations;
    private final S3Client s3Client;
    private final S3Presigner s3Presigner;

    @Value("${spring.cloud.aws.s3.bucket}")
    private String bucketName;

    @Value("${app.storage.cdn-url}")
    private String cdnUrl;

    // Upload ไฟล์ไปยัง S3
    public StoredFile upload(String key, InputStream content, String contentType,
                              long contentLength, Map<String, String> metadata) {
        log.info("Uploading file to S3: {}", key);

        ObjectMetadata objectMetadata = ObjectMetadata.builder()
            .contentType(contentType)
            .contentLength(contentLength)
            .metadata(metadata)
            .build();

        S3Resource resource = s3Operations.upload(bucketName, key, content, objectMetadata);

        String s3Url = resource.getURL().toString();
        String cdnFileUrl = cdnUrl + "/" + key;

        log.info("File uploaded successfully: {}", key);
        return new StoredFile(key, s3Url, cdnFileUrl, contentType, contentLength);
    }

    // Download ไฟล์จาก S3
    public InputStream download(String key) {
        log.info("Downloading file from S3: {}", key);
        return s3Operations.download(bucketName, key);
    }

    // ลบไฟล์
    public void delete(String key) {
        log.info("Deleting file from S3: {}", key);
        s3Operations.deleteObject(bucketName, key);
    }

    // สร้าง Presigned URL สำหรับ upload โดยตรงจาก client
    public PresignedUploadUrl createPresignedUploadUrl(String key, String contentType,
                                                        Duration expiration) {
        log.info("Creating presigned upload URL for: {}", key);

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
            .signatureDuration(expiration)
            .putObjectRequest(r -> r
                .bucket(bucketName)
                .key(key)
                .contentType(contentType)
            )
            .build();

        PresignedPutObjectRequest presignedRequest = s3Presigner.presignPutObject(presignRequest);
        URL presignedUrl = presignedRequest.url();

        return new PresignedUploadUrl(
            presignedUrl.toString(),
            key,
            expiration.toSeconds()
        );
    }

    // สร้าง Presigned URL สำหรับ download (ไฟล์ private)
    public String createPresignedDownloadUrl(String key, Duration expiration) {
        log.info("Creating presigned download URL for: {}", key);

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(expiration)
            .getObjectRequest(r -> r
                .bucket(bucketName)
                .key(key)
            )
            .build();

        return s3Presigner.presignGetObject(presignRequest).url().toString();
    }

    // Copy ไฟล์ภายใน S3 (ไม่ต้อง download/upload ใหม่)
    public void copy(String sourceKey, String destinationKey) {
        s3Client.copyObject(r -> r
            .sourceBucket(bucketName)
            .sourceKey(sourceKey)
            .destinationBucket(bucketName)
            .destinationKey(destinationKey)
        );
    }

    // กำหนด lifecycle rule สำหรับไฟล์ temp
    public void setObjectExpiration(String key, int daysToExpire) {
        s3Client.putObjectTagging(r -> r
            .bucket(bucketName)
            .key(key)
            .tagging(t -> t
                .tagSet(Tag.builder()
                    .key("expires")
                    .value("true")
                    .build())
            )
        );
    }

    // ตรวจสอบว่า file มีอยู่หรือไม่
    public boolean exists(String key) {
        try {
            s3Client.headObject(r -> r.bucket(bucketName).key(key));
            return true;
        } catch (NoSuchKeyException e) {
            return false;
        }
    }

    // Records สำหรับ return types
    public record StoredFile(String key, String s3Url, String cdnUrl,
                              String contentType, long size) {}

    public record PresignedUploadUrl(String url, String key, long expiresInSeconds) {}
}
```

---

## ขั้นตอนที่ 2684: Image Processing Service

```java
// service/ImageProcessingService.java
package com.example.storage.service;

import lombok.extern.slf4j.Slf4j;
import net.coobird.thumbnailator.Thumbnails;
import net.coobird.thumbnailator.geometry.Positions;
import org.springframework.stereotype.Service;

import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import javax.imageio.ImageIO;

@Slf4j
@Service
public class ImageProcessingService {

    // ขนาด thumbnail มาตรฐาน
    public static final int THUMB_SMALL = 150;
    public static final int THUMB_MEDIUM = 400;
    public static final int THUMB_LARGE = 800;

    // Resize รูปภาพ
    public byte[] resize(byte[] imageBytes, int width, int height) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        Thumbnails.of(new ByteArrayInputStream(imageBytes))
            .size(width, height)
            .keepAspectRatio(true)
            .outputQuality(0.85)
            .toOutputStream(output);

        return output.toByteArray();
    }

    // Crop ตรงกลาง
    public byte[] cropCenter(byte[] imageBytes, int width, int height) throws IOException {
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        Thumbnails.of(new ByteArrayInputStream(imageBytes))
            .size(width, height)
            .crop(Positions.CENTER)
            .outputQuality(0.85)
            .toOutputStream(output);

        return output.toByteArray();
    }

    // แปลงเป็น WebP format (ประหยัด bandwidth)
    public byte[] convertToWebP(byte[] imageBytes) throws IOException {
        // ต้องใช้ library เพิ่มเติม เช่น webp-imageio
        // นี่เป็น placeholder
        return imageBytes;
    }

    // สร้าง thumbnails หลายขนาดพร้อมกัน
    public List<ThumbnailResult> generateThumbnails(
            byte[] originalBytes, String baseKey) throws IOException {

        List<ThumbnailResult> results = new ArrayList<>();
        int[][] sizes = {{150, 150}, {400, 400}, {800, 600}};
        String[] suffixes = {"small", "medium", "large"};

        for (int i = 0; i < sizes.length; i++) {
            ByteArrayOutputStream output = new ByteArrayOutputStream();

            Thumbnails.of(new ByteArrayInputStream(originalBytes))
                .size(sizes[i][0], sizes[i][1])
                .keepAspectRatio(true)
                .outputQuality(0.85)
                .outputFormat("jpg")
                .toOutputStream(output);

            String key = baseKey + "_" + suffixes[i] + ".jpg";
            results.add(new ThumbnailResult(key, output.toByteArray(), sizes[i][0], sizes[i][1]));

            log.debug("Generated thumbnail {}: {}x{}", suffixes[i], sizes[i][0], sizes[i][1]);
        }

        return results;
    }

    // ดึงข้อมูล metadata ของรูปภาพ
    public ImageMetadata getMetadata(byte[] imageBytes) throws IOException {
        BufferedImage image = ImageIO.read(new ByteArrayInputStream(imageBytes));
        if (image == null) throw new IOException("ไม่สามารถอ่านไฟล์รูปภาพได้");

        return new ImageMetadata(
            image.getWidth(),
            image.getHeight(),
            imageBytes.length
        );
    }

    // ตรวจสอบ minimum resolution
    public boolean meetsMinimumResolution(byte[] imageBytes, int minWidth, int minHeight)
            throws IOException {
        ImageMetadata meta = getMetadata(imageBytes);
        return meta.width() >= minWidth && meta.height() >= minHeight;
    }

    public record ThumbnailResult(String key, byte[] bytes, int width, int height) {}
    public record ImageMetadata(int width, int height, long sizeBytes) {}
}
```

---

## ขั้นตอนที่ 2685: File Upload Service รวม

```java
// service/FileUploadService.java
package com.example.storage.service;

import com.example.storage.entity.FileRecord;
import com.example.storage.repository.FileRecordRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.time.LocalDateTime;
import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class FileUploadService {

    private final FileValidationService validationService;
    private final S3StorageService storageService;
    private final ImageProcessingService imageProcessingService;
    private final FileRecordRepository fileRecordRepository;

    @Value("${app.storage.cdn-url}")
    private String cdnUrl;

    @Transactional
    public UploadResult uploadFile(MultipartFile file, String userId,
                                    String folder) throws IOException {
        // 1. Validate
        FileValidationService.ValidationResult validation = validationService.validate(file);
        if (!validation.valid()) {
            throw new IllegalArgumentException(validation.message());
        }

        // 2. สร้าง unique key
        String originalFilename = file.getOriginalFilename();
        String extension = getExtension(originalFilename);
        String uniqueKey = folder + "/" + UUID.randomUUID() + "." + extension;

        // 3. Read bytes
        byte[] fileBytes = file.getBytes();

        // 4. Upload ต้นฉบับ
        Map<String, String> metadata = Map.of(
            "original-filename", Objects.requireNonNull(originalFilename),
            "uploaded-by", userId,
            "upload-timestamp", LocalDateTime.now().toString()
        );

        var stored = storageService.upload(
            uniqueKey,
            new ByteArrayInputStream(fileBytes),
            validation.mimeType(),
            fileBytes.length,
            metadata
        );

        // 5. ถ้าเป็นรูปภาพ ให้สร้าง thumbnails
        List<ThumbnailInfo> thumbnails = new ArrayList<>();
        if (validation.mimeType().startsWith("image/")) {
            try {
                String baseKey = uniqueKey.replace("." + extension, "");
                List<ImageProcessingService.ThumbnailResult> thumbResults =
                    imageProcessingService.generateThumbnails(fileBytes, baseKey);

                for (var thumb : thumbResults) {
                    storageService.upload(
                        thumb.key(),
                        new ByteArrayInputStream(thumb.bytes()),
                        "image/jpeg",
                        thumb.bytes().length,
                        Map.of("thumbnail-of", uniqueKey)
                    );
                    thumbnails.add(new ThumbnailInfo(
                        thumb.key(),
                        cdnUrl + "/" + thumb.key(),
                        thumb.width(),
                        thumb.height()
                    ));
                }
            } catch (Exception e) {
                log.warn("Failed to generate thumbnails for {}: {}", uniqueKey, e.getMessage());
            }
        }

        // 6. บันทึก metadata ลง database
        FileRecord fileRecord = FileRecord.builder()
            .key(uniqueKey)
            .originalName(originalFilename)
            .contentType(validation.mimeType())
            .sizeBytes(fileBytes.length)
            .cdnUrl(stored.cdnUrl())
            .uploadedBy(userId)
            .uploadedAt(LocalDateTime.now())
            .build();

        fileRecordRepository.save(fileRecord);

        return new UploadResult(
            fileRecord.getId(),
            uniqueKey,
            stored.cdnUrl(),
            validation.mimeType(),
            fileBytes.length,
            thumbnails
        );
    }

    // Direct upload - client upload ตรงไปยัง S3
    public PresignedUploadInfo createDirectUploadUrl(String filename, String contentType,
                                                      String userId) {
        String key = "uploads/" + userId + "/" + UUID.randomUUID() + "_" + filename;
        var presigned = storageService.createPresignedUploadUrl(
            key, contentType, java.time.Duration.ofMinutes(15)
        );

        return new PresignedUploadInfo(
            presigned.url(),
            presigned.key(),
            presigned.expiresInSeconds()
        );
    }

    // Confirm upload หลังจาก client upload เสร็จ
    @Transactional
    public FileRecord confirmUpload(String key, String userId) {
        if (!storageService.exists(key)) {
            throw new IllegalStateException("ไม่พบไฟล์ใน S3: " + key);
        }

        FileRecord record = FileRecord.builder()
            .key(key)
            .cdnUrl(cdnUrl + "/" + key)
            .uploadedBy(userId)
            .uploadedAt(LocalDateTime.now())
            .build();

        return fileRecordRepository.save(record);
    }

    private String getExtension(String filename) {
        if (filename == null) return "bin";
        int dot = filename.lastIndexOf('.');
        return dot >= 0 ? filename.substring(dot + 1).toLowerCase() : "bin";
    }

    public record UploadResult(
        Long fileId, String key, String url, String contentType,
        long sizeBytes, List<ThumbnailInfo> thumbnails) {}

    public record ThumbnailInfo(String key, String url, int width, int height) {}
    public record PresignedUploadInfo(String uploadUrl, String key, long expiresInSeconds) {}
}
```

---

## ขั้นตอนที่ 2686: Large File Streaming

การ stream ไฟล์ขนาดใหญ่ไม่ควร load ทั้งไฟล์เข้า memory

```java
// controller/FileStreamController.java
package com.example.storage.controller;

import com.example.storage.service.S3StorageService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

import java.io.InputStream;

@Slf4j
@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
public class FileStreamController {

    private final S3StorageService storageService;

    // Stream ไฟล์โดยตรง (ไม่ load เข้า memory ทั้งหมด)
    @GetMapping("/stream/{key}")
    public ResponseEntity<StreamingResponseBody> streamFile(
            @PathVariable String key,
            @RequestHeader(value = "Range", required = false) String rangeHeader) {

        InputStream fileStream = storageService.download(key);

        StreamingResponseBody streamingBody = outputStream -> {
            byte[] buffer = new byte[8192];  // 8KB buffer
            int bytesRead;
            try (fileStream) {
                while ((bytesRead = fileStream.read(buffer)) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                }
            }
        };

        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + key + "\"")
            .body(streamingBody);
    }

    // Download ด้วย InputStreamResource
    @GetMapping("/download/{key}")
    public ResponseEntity<InputStreamResource> downloadFile(@PathVariable String key) {
        InputStream stream = storageService.download(key);
        InputStreamResource resource = new InputStreamResource(stream);

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"file\"")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(resource);
    }

    // สร้าง presigned URL สำหรับ download ตรง
    @GetMapping("/presigned/{key}")
    public ResponseEntity<String> getPresignedUrl(@PathVariable String key) {
        String url = storageService.createPresignedDownloadUrl(
            key, java.time.Duration.ofHours(1)
        );
        return ResponseEntity.ok(url);
    }
}
```

---

## ขั้นตอนที่ 2687: File Upload Controller

```java
// controller/FileUploadController.java
package com.example.storage.controller;

import com.example.storage.service.FileUploadService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping("/api/v1/uploads")
@RequiredArgsConstructor
public class FileUploadController {

    private final FileUploadService fileUploadService;

    // Upload ไฟล์เดียว
    @PostMapping("/single")
    public ResponseEntity<FileUploadService.UploadResult> uploadSingle(
            @RequestParam("file") MultipartFile file,
            @RequestParam(defaultValue = "general") String folder,
            @AuthenticationPrincipal UserDetails user) throws IOException {

        var result = fileUploadService.uploadFile(file, user.getUsername(), folder);
        return ResponseEntity.ok(result);
    }

    // Upload หลายไฟล์พร้อมกัน
    @PostMapping("/multiple")
    public ResponseEntity<List<FileUploadService.UploadResult>> uploadMultiple(
            @RequestParam("files") List<MultipartFile> files,
            @RequestParam(defaultValue = "general") String folder,
            @AuthenticationPrincipal UserDetails user) {

        List<FileUploadService.UploadResult> results = new ArrayList<>();
        List<String> errors = new ArrayList<>();

        for (MultipartFile file : files) {
            try {
                results.add(fileUploadService.uploadFile(file, user.getUsername(), folder));
            } catch (Exception e) {
                errors.add(file.getOriginalFilename() + ": " + e.getMessage());
            }
        }

        if (!errors.isEmpty()) {
            // บางไฟล์ upload สำเร็จ บางไฟล์ไม่สำเร็จ
            return ResponseEntity.status(207).body(results);
        }

        return ResponseEntity.ok(results);
    }

    // ขอ Presigned URL สำหรับ direct upload
    @PostMapping("/presigned")
    public ResponseEntity<FileUploadService.PresignedUploadInfo> getPresignedUrl(
            @RequestParam String filename,
            @RequestParam String contentType,
            @AuthenticationPrincipal UserDetails user) {

        var info = fileUploadService.createDirectUploadUrl(
            filename, contentType, user.getUsername()
        );
        return ResponseEntity.ok(info);
    }

    // ยืนยันว่า direct upload เสร็จแล้ว
    @PostMapping("/confirm")
    public ResponseEntity<?> confirmUpload(
            @RequestParam String key,
            @AuthenticationPrincipal UserDetails user) {

        var record = fileUploadService.confirmUpload(key, user.getUsername());
        return ResponseEntity.ok(record);
    }
}
```

---

## ขั้นตอนที่ 2688: Entity และ Repository

```java
// entity/FileRecord.java
package com.example.storage.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "file_records")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class FileRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String key;  // S3 object key

    @Column(name = "original_name")
    private String originalName;

    @Column(name = "content_type")
    private String contentType;

    @Column(name = "size_bytes")
    private Long sizeBytes;

    @Column(name = "cdn_url")
    private String cdnUrl;

    @Column(name = "uploaded_by")
    private String uploadedBy;

    @Column(name = "uploaded_at")
    private LocalDateTime uploadedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @Column(name = "is_public")
    private boolean isPublic = true;

    // Reference ไปยัง entity อื่น (optional)
    @Column(name = "entity_type")
    private String entityType;

    @Column(name = "entity_id")
    private String entityId;
}
```

```java
// repository/FileRecordRepository.java
package com.example.storage.repository;

import com.example.storage.entity.FileRecord;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface FileRecordRepository extends JpaRepository<FileRecord, Long> {

    Optional<FileRecord> findByKey(String key);

    List<FileRecord> findByUploadedBy(String userId);

    List<FileRecord> findByEntityTypeAndEntityId(String entityType, String entityId);

    @Query("SELECT f FROM FileRecord f WHERE f.deletedAt IS NULL AND f.uploadedAt < :before")
    List<FileRecord> findOrphanedFiles(LocalDateTime before);

    void deleteByKey(String key);
}
```

---

## ขั้นตอนที่ 2689: CDN Configuration และ Cache Headers

```java
// config/CdnConfig.java
package com.example.storage.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.http.CacheControl;
import org.springframework.web.servlet.config.annotation.*;

import java.util.concurrent.TimeUnit;

@Configuration
public class CdnConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // Static files ที่ serve จาก local path
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS).immutable())
            .resourceChain(true);
    }
}
```

```java
// service/CdnService.java
package com.example.storage.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Slf4j
@Service
public class CdnService {

    @Value("${app.storage.cdn-url}")
    private String cdnUrl;

    // Purge cache เมื่อ file เปลี่ยน (ใช้กับ CloudFront)
    public void purgeCache(String... paths) {
        log.info("Purging CDN cache for {} paths", paths.length);
        // สำหรับ CloudFront ต้องใช้ AWS SDK สร้าง invalidation
        // สำหรับ Cloudflare ใช้ API call
    }

    // สร้าง URL ที่มี cache version
    public String versionedUrl(String key, String version) {
        return cdnUrl + "/" + key + "?v=" + version;
    }

    // Responsive image URLs
    public ImageUrls getImageUrls(String baseKey) {
        String base = baseKey.replaceAll("\\.[^.]+$", "");
        return new ImageUrls(
            cdnUrl + "/" + base + "_small.jpg",
            cdnUrl + "/" + base + "_medium.jpg",
            cdnUrl + "/" + base + "_large.jpg",
            cdnUrl + "/" + baseKey
        );
    }

    public record ImageUrls(String small, String medium, String large, String original) {}
}
```

---

## ขั้นตอนที่ 2690: ทดสอบ File Upload

```java
// test/FileUploadControllerTest.java
package com.example.storage.controller;

import com.example.storage.service.FileUploadService;
import com.example.storage.service.FileValidationService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Collections;
import java.util.List;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(FileUploadController.class)
class FileUploadControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private FileUploadService fileUploadService;

    @Test
    @WithMockUser(username = "testuser")
    void shouldUploadFile() throws Exception {
        MockMultipartFile file = new MockMultipartFile(
            "file", "test.jpg", MediaType.IMAGE_JPEG_VALUE,
            "fake image content".getBytes()
        );

        FileUploadService.UploadResult result = new FileUploadService.UploadResult(
            1L, "general/uuid.jpg",
            "https://cdn.example.com/general/uuid.jpg",
            "image/jpeg", 100L, Collections.emptyList()
        );

        when(fileUploadService.uploadFile(any(), anyString(), anyString()))
            .thenReturn(result);

        mockMvc.perform(multipart("/api/v1/uploads/single")
                .file(file)
                .param("folder", "general"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.key").value("general/uuid.jpg"))
            .andExpect(jsonPath("$.url").value("https://cdn.example.com/general/uuid.jpg"));
    }

    @Test
    @WithMockUser(username = "testuser")
    void shouldRejectInvalidFile() throws Exception {
        MockMultipartFile file = new MockMultipartFile(
            "file", "malware.exe", MediaType.APPLICATION_OCTET_STREAM_VALUE,
            "malicious content".getBytes()
        );

        when(fileUploadService.uploadFile(any(), anyString(), anyString()))
            .thenThrow(new IllegalArgumentException("ประเภทไฟล์ไม่ได้รับอนุญาต"));

        mockMvc.perform(multipart("/api/v1/uploads/single").file(file))
            .andExpect(status().isBadRequest());
    }
}
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:

1. **Multipart Upload** - กำหนด configuration และรับ file upload
2. **File Validation** - ตรวจสอบ type ด้วย Apache Tika, ขนาด และชื่อไฟล์
3. **AWS S3 Integration** - Upload/download/delete ด้วย Spring Cloud AWS
4. **Presigned URLs** - Direct upload/download จาก client ตรงไป S3
5. **Image Processing** - Resize, crop, thumbnail generation ด้วย Thumbnailator
6. **Large File Streaming** - Stream ไฟล์โดยไม่ load เข้า memory ทั้งหมด
7. **CDN Integration** - Cache headers และ CDN URL management

---

*[← Part 76: Search with Elasticsearch](./part-76-search-elasticsearch.md) | [Part 78: Email & Notifications →](./part-78-email-notifications.md)*

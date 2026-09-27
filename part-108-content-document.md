# Part 108: โปรเจค 36-40 — Content & Document Services

> **ระดับ:** ระดับโลก | **เวลาเรียนรู้:** 10-15 ชั่วโมง | **โปรเจค:** 36-40

---

## โปรเจค 36: Report Generation Service

### ภาพรวม
บริการสร้างรายงานที่รองรับ Template รายงาน, การผูกข้อมูลแบบ Dynamic (Dynamic Data Binding), Export เป็น PDF และ Excel, การกำหนดเวลาสร้างรายงาน (Scheduled Reports), การส่งทาง Email และประวัติรายงาน

### Dependencies (pom.xml)
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.thymeleaf</groupId>
        <artifactId>thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>com.openhtmltopdf</groupId>
        <artifactId>openhtmltopdf-pdfbox</artifactId>
        <version>1.0.10</version>
    </dependency>
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.4</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
    </dependency>
</dependencies>
```

### Flyway Migration
```sql
-- V1__create_report_tables.sql
CREATE TABLE report_templates (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    template_content TEXT NOT NULL,
    template_type VARCHAR(20) DEFAULT 'HTML',
    data_source_config JSONB,
    parameters_schema JSONB,
    output_format VARCHAR(10) DEFAULT 'PDF',
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE report_schedules (
    id BIGSERIAL PRIMARY KEY,
    template_id BIGINT REFERENCES report_templates(id),
    name VARCHAR(100) NOT NULL,
    cron_expression VARCHAR(100) NOT NULL,
    parameters JSONB,
    recipients JSONB,
    enabled BOOLEAN DEFAULT TRUE,
    last_run_at TIMESTAMP,
    next_run_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE report_history (
    id BIGSERIAL PRIMARY KEY,
    template_id BIGINT REFERENCES report_templates(id),
    schedule_id BIGINT REFERENCES report_schedules(id),
    requested_by BIGINT,
    parameters JSONB,
    output_format VARCHAR(10),
    status VARCHAR(20) DEFAULT 'PENDING',
    file_url VARCHAR(1000),
    file_size BIGINT,
    error_message TEXT,
    generated_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Entity Classes
```java
// ReportTemplate.java
@Entity
@Table(name = "report_templates")
@Data
@NoArgsConstructor
public class ReportTemplate {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(name = "template_content", columnDefinition = "TEXT", nullable = false)
    private String templateContent;

    @Enumerated(EnumType.STRING)
    @Column(name = "template_type")
    private TemplateType templateType = TemplateType.HTML;

    @Type(JsonType.class)
    @Column(name = "data_source_config", columnDefinition = "jsonb")
    private Map<String, Object> dataSourceConfig;

    @Type(JsonType.class)
    @Column(name = "parameters_schema", columnDefinition = "jsonb")
    private List<ParameterSchema> parametersSchema;

    @Enumerated(EnumType.STRING)
    @Column(name = "output_format")
    private OutputFormat outputFormat = OutputFormat.PDF;

    @Column(name = "created_by")
    private Long createdBy;

    public enum TemplateType { HTML, JASPER }
    public enum OutputFormat { PDF, EXCEL, CSV }
}

// ReportHistory.java
@Entity
@Table(name = "report_history")
@Data
@NoArgsConstructor
public class ReportHistory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "template_id")
    private ReportTemplate template;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "schedule_id")
    private ReportSchedule schedule;

    @Column(name = "requested_by")
    private Long requestedBy;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> parameters;

    @Enumerated(EnumType.STRING)
    @Column(name = "output_format")
    private ReportTemplate.OutputFormat outputFormat;

    @Enumerated(EnumType.STRING)
    private ReportStatus status = ReportStatus.PENDING;

    @Column(name = "file_url")
    private String fileUrl;

    @Column(name = "file_size")
    private Long fileSize;

    @Column(name = "error_message")
    private String errorMessage;

    @Column(name = "generated_at")
    private LocalDateTime generatedAt;

    public enum ReportStatus { PENDING, PROCESSING, COMPLETED, FAILED }
}
```

### Service
```java
// ReportGenerationService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ReportGenerationService {

    private final ReportTemplateRepository templateRepository;
    private final ReportHistoryRepository historyRepository;
    private final DataSourceQueryService dataSourceService;
    private final TemplateEngine templateEngine;
    private final S3Service s3Service;
    private final EmailService emailService;

    @Async
    public CompletableFuture<ReportHistory> generateReport(Long templateId,
                                                            Map<String, Object> parameters,
                                                            Long requestedBy) {
        ReportTemplate template = templateRepository.findById(templateId)
                .orElseThrow(() -> new ResourceNotFoundException("Template not found"));

        ReportHistory history = new ReportHistory();
        history.setTemplate(template);
        history.setParameters(parameters);
        history.setOutputFormat(template.getOutputFormat());
        history.setRequestedBy(requestedBy);
        history.setStatus(ReportHistory.ReportStatus.PROCESSING);
        historyRepository.save(history);

        try {
            // Fetch data for report
            Map<String, Object> reportData = dataSourceService.fetchData(
                    template.getDataSourceConfig(), parameters);
            reportData.putAll(parameters);

            byte[] reportBytes;
            String contentType;
            String extension;

            switch (template.getOutputFormat()) {
                case PDF -> {
                    reportBytes = generatePdf(template.getTemplateContent(), reportData);
                    contentType = "application/pdf";
                    extension = "pdf";
                }
                case EXCEL -> {
                    reportBytes = generateExcel(template.getTemplateContent(), reportData);
                    contentType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";
                    extension = "xlsx";
                }
                default -> {
                    reportBytes = generateCsv(reportData);
                    contentType = "text/csv";
                    extension = "csv";
                }
            }

            // Upload to S3
            String fileName = template.getName() + "_" + System.currentTimeMillis() + "." + extension;
            String fileUrl = s3Service.upload(reportBytes, "reports/" + fileName, contentType);

            history.setFileUrl(fileUrl);
            history.setFileSize((long) reportBytes.length);
            history.setStatus(ReportHistory.ReportStatus.COMPLETED);
            history.setGeneratedAt(LocalDateTime.now());

        } catch (Exception e) {
            log.error("Report generation failed: {}", e.getMessage(), e);
            history.setStatus(ReportHistory.ReportStatus.FAILED);
            history.setErrorMessage(e.getMessage());
        }

        historyRepository.save(history);
        return CompletableFuture.completedFuture(history);
    }

    private byte[] generatePdf(String htmlTemplate, Map<String, Object> data) throws Exception {
        // Render HTML with Thymeleaf
        Context context = new Context();
        context.setVariables(data);
        String html = templateEngine.process(htmlTemplate, context);

        // Convert HTML to PDF using OpenHTMLToPDF
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        PdfRendererBuilder builder = new PdfRendererBuilder();
        builder.withHtmlContent(html, null);
        builder.toStream(baos);
        builder.run();
        return baos.toByteArray();
    }

    private byte[] generateExcel(String templateConfig, Map<String, Object> data) throws Exception {
        try (Workbook workbook = new XSSFWorkbook()) {
            Sheet sheet = workbook.createSheet("Report");

            // Create header style
            CellStyle headerStyle = workbook.createCellStyle();
            Font headerFont = workbook.createFont();
            headerFont.setBold(true);
            headerStyle.setFont(headerFont);
            headerStyle.setFillForegroundColor(IndexedColors.LIGHT_BLUE.getIndex());
            headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);

            // Parse template config to determine columns
            List<Map<String, Object>> columns = (List<Map<String, Object>>) parseTemplateConfig(templateConfig).get("columns");
            List<Map<String, Object>> rows = (List<Map<String, Object>>) data.get("rows");

            // Write headers
            Row headerRow = sheet.createRow(0);
            for (int i = 0; i < columns.size(); i++) {
                Cell cell = headerRow.createCell(i);
                cell.setCellValue((String) columns.get(i).get("header"));
                cell.setCellStyle(headerStyle);
                sheet.autoSizeColumn(i);
            }

            // Write data rows
            if (rows != null) {
                for (int rowIdx = 0; rowIdx < rows.size(); rowIdx++) {
                    Row row = sheet.createRow(rowIdx + 1);
                    Map<String, Object> rowData = rows.get(rowIdx);
                    for (int colIdx = 0; colIdx < columns.size(); colIdx++) {
                        String field = (String) columns.get(colIdx).get("field");
                        Object value = rowData.get(field);
                        Cell cell = row.createCell(colIdx);
                        if (value instanceof Number n) {
                            cell.setCellValue(n.doubleValue());
                        } else {
                            cell.setCellValue(value != null ? value.toString() : "");
                        }
                    }
                }
            }

            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            workbook.write(baos);
            return baos.toByteArray();
        }
    }
}
```

### Controller
```java
// ReportController.java
@RestController
@RequestMapping("/api/reports")
@RequiredArgsConstructor
public class ReportController {

    private final ReportGenerationService reportService;

    @PostMapping("/generate/{templateId}")
    public ResponseEntity<ReportHistoryDTO> generateReport(
            @PathVariable Long templateId,
            @RequestBody Map<String, Object> parameters,
            Authentication auth) {
        reportService.generateReport(templateId, parameters, getCurrentUserId(auth));
        return ResponseEntity.accepted().build();
    }

    @GetMapping("/history")
    public ResponseEntity<Page<ReportHistoryDTO>> getHistory(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            Authentication auth) {
        return ResponseEntity.ok(reportService.getUserReportHistory(
                getCurrentUserId(auth), PageRequest.of(page, size)));
    }

    @GetMapping("/history/{id}/download")
    public ResponseEntity<Resource> downloadReport(@PathVariable Long id, Authentication auth) {
        ReportDownload download = reportService.getDownloadUrl(id, getCurrentUserId(auth));
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + download.getFileName() + "\"")
                .contentType(MediaType.parseMediaType(download.getContentType()))
                .body(download.getResource());
    }

    @PostMapping("/templates")
    public ResponseEntity<ReportTemplateDTO> createTemplate(
            @RequestBody @Valid CreateTemplateRequest request, Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ReportTemplateDTO.from(reportService.createTemplate(request, getCurrentUserId(auth))));
    }

    @PostMapping("/schedules")
    public ResponseEntity<ReportScheduleDTO> createSchedule(
            @RequestBody @Valid CreateScheduleRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ReportScheduleDTO.from(reportService.createSchedule(request)));
    }
}
```

---

## โปรเจค 37: PDF Generation Service

### ภาพรวม
บริการสร้าง PDF ที่รองรับ Template ด้วย Thymeleaf/HTML, การรวม PDF (Merge), การเพิ่ม Watermark, Digital Signature, การอัปโหลดไปยัง S3 และ URL ดาวน์โหลดแบบ Presigned

### Flyway Migration
```sql
-- V1__create_pdf_service_tables.sql
CREATE TABLE pdf_templates (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    html_template TEXT NOT NULL,
    page_size VARCHAR(10) DEFAULT 'A4',
    orientation VARCHAR(10) DEFAULT 'PORTRAIT',
    margin_top INTEGER DEFAULT 20,
    margin_bottom INTEGER DEFAULT 20,
    margin_left INTEGER DEFAULT 20,
    margin_right INTEGER DEFAULT 20,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE pdf_jobs (
    id BIGSERIAL PRIMARY KEY,
    job_type VARCHAR(20) NOT NULL,
    template_id BIGINT REFERENCES pdf_templates(id),
    input_data JSONB,
    options JSONB,
    status VARCHAR(20) DEFAULT 'PENDING',
    output_url VARCHAR(1000),
    file_name VARCHAR(255),
    file_size BIGINT,
    error_message TEXT,
    created_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);
```

### Service
```java
// PdfGenerationService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class PdfGenerationService {

    private final PdfTemplateRepository templateRepository;
    private final PdfJobRepository jobRepository;
    private final TemplateEngine templateEngine;
    private final S3Service s3Service;

    public PdfJob generateFromTemplate(Long templateId, Map<String, Object> data, PdfOptions options) {
        PdfTemplate template = templateRepository.findById(templateId)
                .orElseThrow(() -> new ResourceNotFoundException("Template not found"));

        PdfJob job = createJob("GENERATE", templateId, data, options);

        try {
            // Render HTML
            Context context = new Context();
            context.setVariables(data);
            String html = templateEngine.process(template.getHtmlTemplate(), context);

            // Generate PDF
            byte[] pdfBytes = renderHtmlToPdf(html, template, options);

            // Apply watermark if requested
            if (options != null && options.getWatermarkText() != null) {
                pdfBytes = applyWatermark(pdfBytes, options.getWatermarkText());
            }

            // Apply digital signature if requested
            if (options != null && options.isDigitalSignature()) {
                pdfBytes = applyDigitalSignature(pdfBytes, options.getSignatureConfig());
            }

            // Upload to S3
            String fileName = template.getName() + "_" + System.currentTimeMillis() + ".pdf";
            String fileUrl = s3Service.upload(pdfBytes, "pdfs/" + fileName, "application/pdf");

            job.setOutputUrl(fileUrl);
            job.setFileName(fileName);
            job.setFileSize((long) pdfBytes.length);
            job.setStatus(PdfJob.JobStatus.COMPLETED);
            job.setCompletedAt(LocalDateTime.now());

        } catch (Exception e) {
            log.error("PDF generation failed: {}", e.getMessage(), e);
            job.setStatus(PdfJob.JobStatus.FAILED);
            job.setErrorMessage(e.getMessage());
        }

        return jobRepository.save(job);
    }

    public PdfJob mergePdfs(List<String> pdfUrls, MergeOptions options) {
        PdfJob job = createJob("MERGE", null, null, null);

        try {
            PDFMergerUtility merger = new PDFMergerUtility();
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            merger.setDestinationStream(baos);

            for (String url : pdfUrls) {
                byte[] pdfData = s3Service.download(extractKeyFromUrl(url));
                merger.addSource(new ByteArrayInputStream(pdfData));
            }

            merger.mergeDocuments(MemoryUsageSetting.setupMainMemoryOnly());
            byte[] mergedBytes = baos.toByteArray();

            String fileName = "merged_" + System.currentTimeMillis() + ".pdf";
            String fileUrl = s3Service.upload(mergedBytes, "pdfs/merged/" + fileName, "application/pdf");

            job.setOutputUrl(fileUrl);
            job.setFileName(fileName);
            job.setFileSize((long) mergedBytes.length);
            job.setStatus(PdfJob.JobStatus.COMPLETED);
            job.setCompletedAt(LocalDateTime.now());

        } catch (Exception e) {
            job.setStatus(PdfJob.JobStatus.FAILED);
            job.setErrorMessage(e.getMessage());
        }

        return jobRepository.save(job);
    }

    private byte[] applyWatermark(byte[] pdfBytes, String watermarkText) throws Exception {
        try (PDDocument document = PDDocument.load(pdfBytes)) {
            PDFont font = PDType1Font.HELVETICA_BOLD;
            float fontSize = 60;

            for (PDPage page : document) {
                PDRectangle pageSize = page.getMediaBox();
                float stringWidth = font.getStringWidth(watermarkText) / 1000 * fontSize;
                float pageWidth = pageSize.getWidth();
                float pageHeight = pageSize.getHeight();

                try (PDPageContentStream contentStream = new PDPageContentStream(document, page,
                        PDPageContentStream.AppendMode.APPEND, true, true)) {
                    contentStream.setGraphicsStateParameters(
                            buildTransparentState(document, 0.3f));
                    contentStream.beginText();
                    contentStream.setFont(font, fontSize);
                    contentStream.setNonStrokingColor(Color.LIGHT_GRAY);
                    contentStream.setTextMatrix(Matrix.getRotateInstance(
                            Math.toRadians(45),
                            pageWidth / 2 - stringWidth / 2,
                            pageHeight / 2));
                    contentStream.showText(watermarkText);
                    contentStream.endText();
                }
            }

            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            document.save(baos);
            return baos.toByteArray();
        }
    }

    public String getPresignedDownloadUrl(Long jobId, Duration expiry) {
        PdfJob job = jobRepository.findById(jobId)
                .orElseThrow(() -> new ResourceNotFoundException("PDF job not found"));
        if (job.getStatus() != PdfJob.JobStatus.COMPLETED) {
            throw new BusinessException("PDF not ready");
        }
        return s3Service.generatePresignedUrl(extractKeyFromUrl(job.getOutputUrl()), expiry);
    }
}
```

### Controller
```java
// PdfController.java
@RestController
@RequestMapping("/api/pdf")
@RequiredArgsConstructor
public class PdfController {

    private final PdfGenerationService pdfService;

    @PostMapping("/generate/{templateId}")
    public ResponseEntity<PdfJobDTO> generatePdf(
            @PathVariable Long templateId,
            @RequestBody GeneratePdfRequest request) {
        return ResponseEntity.status(HttpStatus.ACCEPTED)
                .body(PdfJobDTO.from(pdfService.generateFromTemplate(
                        templateId, request.getData(), request.getOptions())));
    }

    @PostMapping("/merge")
    public ResponseEntity<PdfJobDTO> mergePdfs(@RequestBody @Valid MergePdfRequest request) {
        return ResponseEntity.status(HttpStatus.ACCEPTED)
                .body(PdfJobDTO.from(pdfService.mergePdfs(request.getPdfUrls(), request.getOptions())));
    }

    @GetMapping("/jobs/{id}/download")
    public ResponseEntity<Void> getDownloadUrl(
            @PathVariable Long id,
            @RequestParam(defaultValue = "3600") long expirySeconds) {
        String presignedUrl = pdfService.getPresignedDownloadUrl(id, Duration.ofSeconds(expirySeconds));
        return ResponseEntity.status(HttpStatus.FOUND)
                .location(URI.create(presignedUrl))
                .build();
    }

    @GetMapping("/jobs/{id}")
    public ResponseEntity<PdfJobDTO> getJobStatus(@PathVariable Long id) {
        return ResponseEntity.ok(PdfJobDTO.from(pdfService.getJob(id)));
    }
}
```

---

## โปรเจค 38: Image Processing Service

### ภาพรวม
บริการประมวลผลภาพที่รองรับการอัปโหลด, ปรับขนาด (Resize), ครอบภาพ (Crop), บีบอัด (Compress), แปลงรูปแบบ (JPEG/PNG/WebP), สร้าง Thumbnail, CDN URL และการประมวลผลแบบ Asynchronous

### Flyway Migration
```sql
-- V1__create_image_service_tables.sql
CREATE TABLE images (
    id BIGSERIAL PRIMARY KEY,
    original_key VARCHAR(500) NOT NULL,
    original_url VARCHAR(1000) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    original_size BIGINT NOT NULL,
    width INTEGER,
    height INTEGER,
    format VARCHAR(10),
    mime_type VARCHAR(50),
    owner_id BIGINT,
    tags JSONB,
    cdn_url VARCHAR(1000),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE image_variants (
    id BIGSERIAL PRIMARY KEY,
    image_id BIGINT REFERENCES images(id) ON DELETE CASCADE,
    variant_name VARCHAR(50) NOT NULL,
    s3_key VARCHAR(500) NOT NULL,
    url VARCHAR(1000) NOT NULL,
    cdn_url VARCHAR(1000),
    width INTEGER,
    height INTEGER,
    file_size BIGINT,
    format VARCHAR(10),
    quality INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(image_id, variant_name)
);

CREATE TABLE image_processing_jobs (
    id BIGSERIAL PRIMARY KEY,
    image_id BIGINT REFERENCES images(id),
    operations JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    result_key VARCHAR(500),
    result_url VARCHAR(1000),
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);
```

### Service
```java
// ImageProcessingService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class ImageProcessingService {

    private final ImageRepository imageRepository;
    private final ImageVariantRepository variantRepository;
    private final S3Service s3Service;
    private final CdnService cdnService;

    private static final Map<String, int[]> THUMBNAIL_SIZES = Map.of(
            "thumbnail", new int[]{150, 150},
            "small", new int[]{400, 300},
            "medium", new int[]{800, 600},
            "large", new int[]{1200, 900}
    );

    public Image uploadImage(MultipartFile file, Long ownerId) throws Exception {
        // Read image metadata
        BufferedImage bufferedImage = ImageIO.read(file.getInputStream());
        if (bufferedImage == null) {
            throw new BusinessException("Invalid image file");
        }

        // Upload original to S3
        String key = "images/original/" + UUID.randomUUID() + "/" + file.getOriginalFilename();
        String s3Url = s3Service.upload(file.getBytes(), key, file.getContentType());
        String cdnUrl = cdnService.getCdnUrl(key);

        Image image = new Image();
        image.setOriginalKey(key);
        image.setOriginalUrl(s3Url);
        image.setCdnUrl(cdnUrl);
        image.setOriginalFilename(file.getOriginalFilename());
        image.setOriginalSize(file.getSize());
        image.setWidth(bufferedImage.getWidth());
        image.setHeight(bufferedImage.getHeight());
        image.setMimeType(file.getContentType());
        image.setFormat(getFormatFromContentType(file.getContentType()));
        image.setOwnerId(ownerId);

        Image saved = imageRepository.save(image);

        // Generate thumbnails asynchronously
        generateThumbnailsAsync(saved.getId());

        return saved;
    }

    @Async
    public void generateThumbnailsAsync(Long imageId) {
        try {
            Image image = imageRepository.findById(imageId)
                    .orElseThrow(() -> new ResourceNotFoundException("Image not found"));
            byte[] originalBytes = s3Service.download(image.getOriginalKey());
            BufferedImage original = ImageIO.read(new ByteArrayInputStream(originalBytes));

            for (Map.Entry<String, int[]> entry : THUMBNAIL_SIZES.entrySet()) {
                generateVariant(image, original, entry.getKey(),
                        entry.getValue()[0], entry.getValue()[1], "JPEG", 85);
            }
        } catch (Exception e) {
            log.error("Failed to generate thumbnails for image {}: {}", imageId, e.getMessage());
        }
    }

    public ImageVariant resizeImage(Long imageId, int width, int height, String format, int quality) throws Exception {
        Image image = imageRepository.findById(imageId)
                .orElseThrow(() -> new ResourceNotFoundException("Image not found"));

        byte[] originalBytes = s3Service.download(image.getOriginalKey());
        BufferedImage original = ImageIO.read(new ByteArrayInputStream(originalBytes));

        String variantName = width + "x" + height + "_" + format.toLowerCase() + "_q" + quality;
        return generateVariant(image, original, variantName, width, height, format, quality);
    }

    public byte[] cropImage(Long imageId, int x, int y, int width, int height) throws Exception {
        Image image = imageRepository.findById(imageId)
                .orElseThrow(() -> new ResourceNotFoundException("Image not found"));

        byte[] originalBytes = s3Service.download(image.getOriginalKey());
        BufferedImage original = ImageIO.read(new ByteArrayInputStream(originalBytes));

        BufferedImage cropped = original.getSubimage(x, y, width, height);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ImageIO.write(cropped, image.getFormat(), baos);
        return baos.toByteArray();
    }

    public byte[] convertFormat(Long imageId, String targetFormat, int quality) throws Exception {
        Image image = imageRepository.findById(imageId)
                .orElseThrow(() -> new ResourceNotFoundException("Image not found"));

        byte[] originalBytes = s3Service.download(image.getOriginalKey());
        BufferedImage bufferedImage = ImageIO.read(new ByteArrayInputStream(originalBytes));

        if ("WEBP".equalsIgnoreCase(targetFormat)) {
            return convertToWebP(bufferedImage, quality);
        }

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ImageWriter writer = ImageIO.getImageWritersByFormatName(targetFormat).next();
        ImageWriteParam params = writer.getDefaultWriteParam();
        if (params.canWriteCompressed()) {
            params.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
            params.setCompressionQuality(quality / 100.0f);
        }
        writer.setOutput(ImageIO.createImageOutputStream(baos));
        writer.write(null, new IIOImage(bufferedImage, null, null), params);

        return baos.toByteArray();
    }

    private ImageVariant generateVariant(Image image, BufferedImage original,
                                          String variantName, int width, int height,
                                          String format, int quality) throws Exception {
        // Scale image maintaining aspect ratio
        BufferedImage scaled = scaleImage(original, width, height);

        // Compress and encode
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        if ("WEBP".equalsIgnoreCase(format)) {
            baos.write(convertToWebP(scaled, quality));
        } else {
            ImageWriter writer = ImageIO.getImageWritersByFormatName(format).next();
            ImageWriteParam params = writer.getDefaultWriteParam();
            if (params.canWriteCompressed()) {
                params.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);
                params.setCompressionQuality(quality / 100.0f);
            }
            ImageOutputStream ios = ImageIO.createImageOutputStream(baos);
            writer.setOutput(ios);
            writer.write(null, new IIOImage(scaled, null, null), params);
            ios.close();
        }

        byte[] variantBytes = baos.toByteArray();
        String extension = format.toLowerCase();
        String key = "images/variants/" + image.getId() + "/" + variantName + "." + extension;
        String url = s3Service.upload(variantBytes, key, "image/" + extension);
        String cdnUrl = cdnService.getCdnUrl(key);

        ImageVariant variant = new ImageVariant();
        variant.setImage(image);
        variant.setVariantName(variantName);
        variant.setS3Key(key);
        variant.setUrl(url);
        variant.setCdnUrl(cdnUrl);
        variant.setWidth(scaled.getWidth());
        variant.setHeight(scaled.getHeight());
        variant.setFileSize((long) variantBytes.length);
        variant.setFormat(format);
        variant.setQuality(quality);

        return variantRepository.save(variant);
    }
}
```

### Controller
```java
// ImageController.java
@RestController
@RequestMapping("/api/images")
@RequiredArgsConstructor
public class ImageController {

    private final ImageProcessingService imageService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ImageDTO> uploadImage(
            @RequestParam("file") MultipartFile file,
            Authentication auth) throws Exception {
        Image image = imageService.uploadImage(file, getCurrentUserId(auth));
        return ResponseEntity.status(HttpStatus.CREATED).body(ImageDTO.from(image));
    }

    @PostMapping("/{id}/resize")
    public ResponseEntity<ImageVariantDTO> resizeImage(
            @PathVariable Long id,
            @RequestBody @Valid ResizeRequest request) throws Exception {
        return ResponseEntity.ok(ImageVariantDTO.from(
                imageService.resizeImage(id, request.getWidth(), request.getHeight(),
                        request.getFormat(), request.getQuality())));
    }

    @GetMapping("/{id}/crop")
    public ResponseEntity<byte[]> cropImage(
            @PathVariable Long id,
            @RequestParam int x,
            @RequestParam int y,
            @RequestParam int width,
            @RequestParam int height) throws Exception {
        byte[] cropped = imageService.cropImage(id, x, y, width, height);
        return ResponseEntity.ok()
                .contentType(MediaType.IMAGE_JPEG)
                .body(cropped);
    }

    @PostMapping("/{id}/convert")
    public ResponseEntity<byte[]> convertFormat(
            @PathVariable Long id,
            @RequestParam String format,
            @RequestParam(defaultValue = "85") int quality) throws Exception {
        byte[] converted = imageService.convertFormat(id, format, quality);
        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType("image/" + format.toLowerCase()))
                .body(converted);
    }

    @GetMapping("/{id}/variants")
    public ResponseEntity<List<ImageVariantDTO>> getVariants(@PathVariable Long id) {
        return ResponseEntity.ok(imageService.getImageVariants(id));
    }
}
```

---

## โปรเจค 39: File Sharing Platform

### ภาพรวม
แพลตฟอร์มแชร์ไฟล์ที่รองรับการอัปโหลดไฟล์, ลิงก์แชร์ (แบบสาธารณะ/ป้องกันรหัสผ่าน/กำหนดอายุ), การติดตามการดาวน์โหลด, โควต้าพื้นที่จัดเก็บ (Storage Quota) และการจัดระเบียบด้วยโฟลเดอร์

### Flyway Migration
```sql
-- V1__create_file_sharing_tables.sql
CREATE TABLE storage_folders (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    owner_id BIGINT NOT NULL,
    parent_id BIGINT REFERENCES storage_folders(id),
    path VARCHAR(2000),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stored_files (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    original_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100),
    size BIGINT NOT NULL,
    s3_key VARCHAR(1000) NOT NULL,
    owner_id BIGINT NOT NULL,
    folder_id BIGINT REFERENCES storage_folders(id),
    checksum VARCHAR(64),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sharing_links (
    id BIGSERIAL PRIMARY KEY,
    file_id BIGINT REFERENCES stored_files(id) ON DELETE CASCADE,
    token VARCHAR(64) UNIQUE NOT NULL,
    access_type VARCHAR(20) DEFAULT 'PUBLIC',
    password_hash VARCHAR(255),
    max_downloads INTEGER,
    download_count INTEGER DEFAULT 0,
    expires_at TIMESTAMP,
    created_by BIGINT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE download_logs (
    id BIGSERIAL PRIMARY KEY,
    link_id BIGINT REFERENCES sharing_links(id),
    file_id BIGINT REFERENCES stored_files(id),
    ip_address VARCHAR(45),
    user_agent TEXT,
    downloaded_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE storage_quotas (
    user_id BIGINT PRIMARY KEY,
    total_quota BIGINT DEFAULT 1073741824,
    used_space BIGINT DEFAULT 0,
    file_count INTEGER DEFAULT 0
);
```

### Service
```java
// FileSharingService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class FileSharingService {

    private final StoredFileRepository fileRepository;
    private final SharingLinkRepository linkRepository;
    private final DownloadLogRepository downloadLogRepository;
    private final StorageQuotaRepository quotaRepository;
    private final S3Service s3Service;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public StoredFile uploadFile(MultipartFile file, Long ownerId, Long folderId) throws Exception {
        // Check quota
        StorageQuota quota = quotaRepository.findByUserId(ownerId)
                .orElseGet(() -> createDefaultQuota(ownerId));

        if (quota.getUsedSpace() + file.getSize() > quota.getTotalQuota()) {
            throw new BusinessException("Storage quota exceeded");
        }

        // Upload to S3
        String key = "files/" + ownerId + "/" + UUID.randomUUID() + "/" + file.getOriginalFilename();
        String checksum = calculateChecksum(file.getBytes());
        s3Service.upload(file.getBytes(), key, file.getContentType());

        StoredFile storedFile = new StoredFile();
        storedFile.setName(file.getOriginalFilename());
        storedFile.setOriginalName(file.getOriginalFilename());
        storedFile.setMimeType(file.getContentType());
        storedFile.setSize(file.getSize());
        storedFile.setS3Key(key);
        storedFile.setOwnerId(ownerId);
        storedFile.setChecksum(checksum);
        if (folderId != null) {
            storedFile.setFolder(new StorageFolder(folderId));
        }

        StoredFile saved = fileRepository.save(storedFile);

        // Update quota
        quota.setUsedSpace(quota.getUsedSpace() + file.getSize());
        quota.setFileCount(quota.getFileCount() + 1);
        quotaRepository.save(quota);

        return saved;
    }

    public SharingLink createSharingLink(Long fileId, CreateLinkRequest request, Long createdBy) {
        StoredFile file = fileRepository.findById(fileId)
                .orElseThrow(() -> new ResourceNotFoundException("File not found"));

        if (!file.getOwnerId().equals(createdBy)) {
            throw new AccessDeniedException("Not authorized to share this file");
        }

        SharingLink link = new SharingLink();
        link.setFile(file);
        link.setToken(generateSecureToken());
        link.setAccessType(request.getAccessType());
        link.setExpiresAt(request.getExpiresAt());
        link.setMaxDownloads(request.getMaxDownloads());
        link.setCreatedBy(createdBy);

        if ("PASSWORD_PROTECTED".equals(request.getAccessType()) && request.getPassword() != null) {
            link.setPasswordHash(passwordEncoder.encode(request.getPassword()));
        }

        return linkRepository.save(link);
    }

    public Resource downloadByToken(String token, String password, HttpServletRequest httpRequest) {
        SharingLink link = linkRepository.findByToken(token)
                .orElseThrow(() -> new ResourceNotFoundException("Sharing link not found"));

        // Validate link
        if (!link.getIsActive()) throw new BusinessException("Link is no longer active");
        if (link.getExpiresAt() != null && LocalDateTime.now().isAfter(link.getExpiresAt())) {
            throw new BusinessException("Link has expired");
        }
        if (link.getMaxDownloads() != null && link.getDownloadCount() >= link.getMaxDownloads()) {
            throw new BusinessException("Download limit reached");
        }
        if ("PASSWORD_PROTECTED".equals(link.getAccessType())) {
            if (password == null || !passwordEncoder.matches(password, link.getPasswordHash())) {
                throw new AccessDeniedException("Invalid password");
            }
        }

        // Log download
        DownloadLog log = new DownloadLog();
        log.setLink(link);
        log.setFile(link.getFile());
        log.setIpAddress(extractIpAddress(httpRequest));
        log.setUserAgent(httpRequest.getHeader("User-Agent"));
        downloadLogRepository.save(log);

        // Increment download count
        linkRepository.incrementDownloadCount(link.getId());

        // Get file from S3
        byte[] fileBytes = s3Service.download(link.getFile().getS3Key());
        return new ByteArrayResource(fileBytes);
    }

    private String generateSecureToken() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

### Controller
```java
// FileSharingController.java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
public class FileSharingController {

    private final FileSharingService fileService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<FileDTO> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam(required = false) Long folderId,
            Authentication auth) throws Exception {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(FileDTO.from(fileService.uploadFile(file, getCurrentUserId(auth), folderId)));
    }

    @PostMapping("/{id}/share")
    public ResponseEntity<SharingLinkDTO> createSharingLink(
            @PathVariable Long id,
            @RequestBody @Valid CreateLinkRequest request,
            Authentication auth) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(SharingLinkDTO.from(fileService.createSharingLink(id, request, getCurrentUserId(auth))));
    }

    @GetMapping("/shared/{token}")
    public ResponseEntity<Resource> downloadSharedFile(
            @PathVariable String token,
            @RequestParam(required = false) String password,
            HttpServletRequest request) {
        Resource resource = fileService.downloadByToken(token, password, request);
        StoredFile file = fileService.getFileByToken(token);
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + file.getOriginalName() + "\"")
                .contentType(MediaType.parseMediaType(file.getMimeType()))
                .body(resource);
    }

    @GetMapping("/quota")
    public ResponseEntity<StorageQuotaDTO> getQuota(Authentication auth) {
        return ResponseEntity.ok(fileService.getUserQuota(getCurrentUserId(auth)));
    }

    @GetMapping
    public ResponseEntity<List<FileDTO>> listFiles(
            @RequestParam(required = false) Long folderId,
            Authentication auth) {
        return ResponseEntity.ok(fileService.listUserFiles(getCurrentUserId(auth), folderId));
    }
}
```

---

## โปรเจค 40: Document Management System

### ภาพรวม
ระบบจัดการเอกสารครบวงจรที่รองรับเอกสาร, เวอร์ชัน (Versions), Metadata, Full-text Search ด้วย Elasticsearch, การควบคุมการเข้าถึง (Access Control), การ Checkout/Checkin และ Approval Workflow

### Flyway Migration
```sql
-- V1__create_document_management_tables.sql
CREATE TABLE document_categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id BIGINT REFERENCES document_categories(id),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    category_id BIGINT REFERENCES document_categories(id),
    owner_id BIGINT NOT NULL,
    status VARCHAR(20) DEFAULT 'DRAFT',
    current_version INTEGER DEFAULT 0,
    is_locked BOOLEAN DEFAULT FALSE,
    locked_by BIGINT,
    locked_at TIMESTAMP,
    tags JSONB,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE document_versions (
    id BIGSERIAL PRIMARY KEY,
    document_id BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    s3_key VARCHAR(1000) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(100),
    checksum VARCHAR(64),
    change_summary TEXT,
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(document_id, version_number)
);

CREATE TABLE document_permissions (
    id BIGSERIAL PRIMARY KEY,
    document_id BIGINT REFERENCES documents(id) ON DELETE CASCADE,
    principal_type VARCHAR(20) NOT NULL,
    principal_id BIGINT NOT NULL,
    permission VARCHAR(20) NOT NULL,
    granted_by BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(document_id, principal_type, principal_id)
);

CREATE TABLE document_approvals (
    id BIGSERIAL PRIMARY KEY,
    document_id BIGINT REFERENCES documents(id),
    version_id BIGINT REFERENCES document_versions(id),
    approver_id BIGINT NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    comment TEXT,
    approved_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Service
```java
// DocumentService.java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class DocumentService {

    private final DocumentRepository documentRepository;
    private final DocumentVersionRepository versionRepository;
    private final DocumentPermissionRepository permissionRepository;
    private final DocumentApprovalRepository approvalRepository;
    private final ElasticsearchClient elasticsearchClient;
    private final S3Service s3Service;

    public Document createDocument(CreateDocumentRequest request, MultipartFile file, Long ownerId) throws Exception {
        Document document = new Document();
        document.setTitle(request.getTitle());
        document.setDescription(request.getDescription());
        document.setOwnerId(ownerId);
        document.setStatus(Document.DocumentStatus.DRAFT);
        document.setTags(request.getTags());
        document.setMetadata(request.getMetadata());

        Document saved = documentRepository.save(document);

        // Create first version
        if (file != null) {
            addVersion(saved.getId(), file, "Initial version", ownerId);
        }

        // Index in Elasticsearch
        indexDocument(saved);

        return saved;
    }

    public DocumentVersion addVersion(Long documentId, MultipartFile file,
                                       String changeSummary, Long createdBy) throws Exception {
        Document document = documentRepository.findById(documentId)
                .orElseThrow(() -> new ResourceNotFoundException("Document not found"));

        // Check if locked by someone else
        if (document.getIsLocked() && !document.getLockedBy().equals(createdBy)) {
            throw new BusinessException("Document is locked by another user");
        }

        // Upload file
        String key = "documents/" + documentId + "/v" + (document.getCurrentVersion() + 1)
                + "/" + file.getOriginalFilename();
        s3Service.upload(file.getBytes(), key, file.getContentType());

        DocumentVersion version = new DocumentVersion();
        version.setDocument(document);
        version.setVersionNumber(document.getCurrentVersion() + 1);
        version.setS3Key(key);
        version.setFileName(file.getOriginalFilename());
        version.setFileSize(file.getSize());
        version.setMimeType(file.getContentType());
        version.setChecksum(calculateChecksum(file.getBytes()));
        version.setChangeSummary(changeSummary);
        version.setCreatedBy(createdBy);

        DocumentVersion saved = versionRepository.save(version);

        // Update document
        document.setCurrentVersion(version.getVersionNumber());
        document.setIsLocked(false);
        document.setLockedBy(null);
        documentRepository.save(document);

        // Re-index in Elasticsearch
        indexDocument(document);

        return saved;
    }

    public Document checkoutDocument(Long documentId, Long userId) {
        Document document = documentRepository.findById(documentId)
                .orElseThrow(() -> new ResourceNotFoundException("Document not found"));

        if (document.getIsLocked()) {
            throw new BusinessException("Document already checked out by user " + document.getLockedBy());
        }

        document.setIsLocked(true);
        document.setLockedBy(userId);
        document.setLockedAt(LocalDateTime.now());

        return documentRepository.save(document);
    }

    public List<DocumentSearchResult> searchDocuments(String query, Long userId) {
        try {
            SearchResponse<DocumentIndex> response = elasticsearchClient.search(s -> s
                    .index("documents")
                    .query(q -> q
                            .multiMatch(mm -> mm
                                    .fields("title^3", "description^2", "content")
                                    .query(query)
                                    .fuzziness("AUTO"))),
                    DocumentIndex.class);

            return response.hits().hits().stream()
                    .map(hit -> new DocumentSearchResult(
                            hit.source(), hit.score(), hit.highlight()))
                    .collect(Collectors.toList());
        } catch (IOException e) {
            throw new RuntimeException("Search failed", e);
        }
    }

    public void submitForApproval(Long documentId, List<Long> approverIds, Long submittedBy) {
        Document document = documentRepository.findById(documentId)
                .orElseThrow(() -> new ResourceNotFoundException("Document not found"));

        DocumentVersion latestVersion = versionRepository
                .findByDocumentIdAndVersionNumber(documentId, document.getCurrentVersion())
                .orElseThrow(() -> new ResourceNotFoundException("Version not found"));

        document.setStatus(Document.DocumentStatus.PENDING_APPROVAL);
        documentRepository.save(document);

        approverIds.forEach(approverId -> {
            DocumentApproval approval = new DocumentApproval();
            approval.setDocument(document);
            approval.setVersion(latestVersion);
            approval.setApproverId(approverId);
            approval.setStatus(DocumentApproval.ApprovalStatus.PENDING);
            approvalRepository.save(approval);
        });
    }

    private void indexDocument(Document document) {
        try {
            DocumentIndex docIndex = new DocumentIndex(
                    document.getId().toString(),
                    document.getTitle(),
                    document.getDescription(),
                    document.getOwnerId(),
                    document.getStatus().name(),
                    document.getTags());

            elasticsearchClient.index(i -> i
                    .index("documents")
                    .id(document.getId().toString())
                    .document(docIndex));
        } catch (IOException e) {
            log.error("Failed to index document {}: {}", document.getId(), e.getMessage());
        }
    }
}
```

### Controller
```java
// DocumentController.java
@RestController
@RequestMapping("/api/documents")
@RequiredArgsConstructor
public class DocumentController {

    private final DocumentService documentService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<DocumentDTO> createDocument(
            @RequestPart("data") @Valid CreateDocumentRequest request,
            @RequestPart(value = "file", required = false) MultipartFile file,
            Authentication auth) throws Exception {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(DocumentDTO.from(documentService.createDocument(request, file, getCurrentUserId(auth))));
    }

    @PostMapping(value = "/{id}/versions", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<DocumentVersionDTO> addVersion(
            @PathVariable Long id,
            @RequestPart("file") MultipartFile file,
            @RequestPart(value = "changeSummary", required = false) String changeSummary,
            Authentication auth) throws Exception {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(DocumentVersionDTO.from(
                        documentService.addVersion(id, file, changeSummary, getCurrentUserId(auth))));
    }

    @PostMapping("/{id}/checkout")
    public ResponseEntity<DocumentDTO> checkoutDocument(
            @PathVariable Long id, Authentication auth) {
        return ResponseEntity.ok(DocumentDTO.from(
                documentService.checkoutDocument(id, getCurrentUserId(auth))));
    }

    @PostMapping("/{id}/checkin")
    public ResponseEntity<DocumentDTO> checkinDocument(
            @PathVariable Long id, Authentication auth) {
        return ResponseEntity.ok(DocumentDTO.from(
                documentService.checkinDocument(id, getCurrentUserId(auth))));
    }

    @GetMapping("/search")
    public ResponseEntity<List<DocumentSearchResult>> searchDocuments(
            @RequestParam String q, Authentication auth) {
        return ResponseEntity.ok(documentService.searchDocuments(q, getCurrentUserId(auth)));
    }

    @PostMapping("/{id}/submit-approval")
    public ResponseEntity<Void> submitForApproval(
            @PathVariable Long id,
            @RequestBody List<Long> approverIds,
            Authentication auth) {
        documentService.submitForApproval(id, approverIds, getCurrentUserId(auth));
        return ResponseEntity.accepted().build();
    }

    @PostMapping("/{id}/approve")
    public ResponseEntity<Void> approveDocument(
            @PathVariable Long id,
            @RequestBody ApprovalRequest request,
            Authentication auth) {
        documentService.processApproval(id, getCurrentUserId(auth), request.isApproved(), request.getComment());
        return ResponseEntity.ok().build();
    }
}
```

### Docker Compose
```yaml
# docker-compose.yml (Part 108)
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/contentdb
      SPRING_ELASTICSEARCH_URIS: http://elasticsearch:9200
      AWS_S3_BUCKET: content-bucket
      AWS_REGION: ap-southeast-1
    depends_on:
      - postgres
      - elasticsearch

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: contentdb
      POSTGRES_USER: content
      POSTGRES_PASSWORD: contentpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      SERVICES: s3
      DEFAULT_REGION: ap-southeast-1

volumes:
  postgres_data:
  es_data:
```

---

## สรุป Part 108

| โปรเจค | เทคโนโลยีหลัก | ความซับซ้อน |
|--------|--------------|------------|
| 36. Report Generation | Thymeleaf, POI, S3, Scheduler | สูง |
| 37. PDF Generation | PDFBox, OpenHTMLToPDF, Watermark | สูง |
| 38. Image Processing | BufferedImage, WebP, S3, CDN | สูง |
| 39. File Sharing | Quota, HMAC Links, S3 | กลาง |
| 40. Document Management | Elasticsearch, Versioning, Workflow | สูงมาก |

---

## Navigation

- [← Part 107: Developer Tools](part-107-developer-tools.md)
- [Part 109: Data & Aggregation →](part-109-data-aggregation.md)
- [กลับหน้าหลัก](README.md)

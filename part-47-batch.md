# Part 47: Spring Batch - Advanced Batch Processing
## ขั้นตอนที่ 1481-1520

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Process large datasets efficiently

---

## ขั้นตอนที่ 1481: Spring Batch Architecture

```
Spring Batch Concepts:
  Job      = ชุดของ Steps
  Step     = หน่วยงานเดียว (มีหลายประเภท)
  ItemReader  = อ่านข้อมูล
  ItemProcessor = transform/filter ข้อมูล
  ItemWriter  = บันทึกข้อมูล
  Chunk    = จำนวน items ที่ process ต่อ transaction
  
Chunk-Oriented Processing:
  Read 100 → Process 100 → Write 100 → Commit
  Read 100 → Process 100 → Write 100 → Commit
  ...repeat until done
  
JobRepository = เก็บ job execution state
JobLauncher  = รัน job
```

---

## ขั้นตอนที่ 1482: Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 1483: Complete Import Job

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
public class ProductImportJobConfig {
    
    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final DataSource dataSource;
    private final ProductRepository productRepository;
    
    // ===== Job =====
    
    @Bean
    public Job productImportJob(Step validateStep, Step importStep, Step cleanupStep) {
        return new JobBuilder("productImportJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .validator(new DefaultJobParametersValidator(
                new String[]{"file.path"},
                new String[]{"chunk.size"}
            ))
            .start(validateStep)
            .next(importStep)
            .next(cleanupStep)
            .listener(jobExecutionListener())
            .build();
    }
    
    // ===== Step 1: Validate =====
    
    @Bean
    public Step validateStep(TaskletStep validateTasklet) {
        return new StepBuilder("validateStep", jobRepository)
            .tasklet(validateTasklet, transactionManager)
            .build();
    }
    
    @Bean
    @StepScope
    public Tasklet validateFileTasklet(@Value("#{jobParameters['file.path']}") String filePath) {
        return (contribution, chunkContext) -> {
            File file = new File(filePath);
            if (!file.exists()) {
                throw new JobParametersInvalidException("File not found: " + filePath);
            }
            log.info("File validated: {} ({} bytes)", filePath, file.length());
            return RepeatStatus.FINISHED;
        };
    }
    
    // ===== Step 2: Import =====
    
    @Bean
    public Step importStep() {
        return new StepBuilder("importStep", jobRepository)
            .<ProductCsvRecord, Product>chunk(100, transactionManager)
            .reader(csvReader(null))
            .processor(productProcessor())
            .writer(productWriter())
            
            // Skip bad records (don't fail job)
            .faultTolerant()
            .skip(ValidationException.class)
            .skip(DataIntegrityViolationException.class)
            .skipLimit(1000)
            
            // Retry transient failures
            .retry(TransientDataAccessException.class)
            .retryLimit(3)
            
            .listener(stepExecutionListener())
            .build();
    }
    
    // ===== Reader =====
    
    @Bean
    @StepScope
    public FlatFileItemReader<ProductCsvRecord> csvReader(
        @Value("#{jobParameters['file.path']}") String filePath
    ) {
        return new FlatFileItemReaderBuilder<ProductCsvRecord>()
            .name("productCsvReader")
            .resource(new FileSystemResource(filePath))
            .delimited()
            .delimiter(",")
            .names("name", "price", "stock", "category", "description")
            .linesToSkip(1)  // Skip header
            .fieldSetMapper(new BeanWrapperFieldSetMapper<>() {{
                setTargetType(ProductCsvRecord.class);
            }})
            .encoding("UTF-8")
            .build();
    }
    
    // ===== Processor =====
    
    @Bean
    public ItemProcessor<ProductCsvRecord, Product> productProcessor() {
        return record -> {
            // Validate
            if (record.name() == null || record.name().isBlank()) {
                throw new ValidationException("Product name is required");
            }
            if (record.price().compareTo(BigDecimal.ZERO) <= 0) {
                throw new ValidationException("Price must be positive: " + record.name());
            }
            
            // Skip already existing products
            if (productRepository.existsByName(record.name())) {
                log.debug("Skipping existing product: {}", record.name());
                return null;  // null = skip this item
            }
            
            return Product.builder()
                .name(record.name())
                .price(record.price())
                .stock(record.stock())
                .status(ProductStatus.ACTIVE)
                .build();
        };
    }
    
    // ===== Writer =====
    
    @Bean
    public JpaItemWriter<Product> productWriter() {
        JpaItemWriter<Product> writer = new JpaItemWriter<>();
        writer.setEntityManagerFactory(entityManagerFactory);
        return writer;
    }
    
    // ===== Step 3: Cleanup =====
    
    @Bean
    public Step cleanupStep() {
        return new StepBuilder("cleanupStep", jobRepository)
            .tasklet((contribution, chunkContext) -> {
                // Archive processed file
                String filePath = (String) chunkContext.getStepContext()
                    .getJobParameters().get("file.path");
                
                Path source = Paths.get(filePath);
                Path dest = source.getParent().resolve("processed/" + source.getFileName());
                Files.move(source, dest, StandardCopyOption.REPLACE_EXISTING);
                
                return RepeatStatus.FINISHED;
            }, transactionManager)
            .build();
    }
    
    // ===== Listeners =====
    
    @Bean
    public JobExecutionListener jobExecutionListener() {
        return new JobExecutionListenerSupport() {
            
            @Override
            public void afterJob(JobExecution jobExecution) {
                if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
                    long readCount = jobExecution.getStepExecutions().stream()
                        .mapToLong(StepExecution::getReadCount)
                        .sum();
                    long writeCount = jobExecution.getStepExecutions().stream()
                        .mapToLong(StepExecution::getWriteCount)
                        .sum();
                    
                    log.info("Import completed: read={}, written={}, skipped={}",
                        readCount, writeCount, readCount - writeCount);
                } else {
                    log.error("Import failed: {}", jobExecution.getStatus());
                }
            }
        };
    }
}
```

---

## ขั้นตอนที่ 1484: Partitioned Step (Parallel Processing)

```java
// Split large file into partitions, process in parallel
@Bean
public Step partitionedImportStep(Step workerStep) {
    return new StepBuilder("partitionedImportStep", jobRepository)
        .partitioner("workerStep", partitioner())
        .step(workerStep)
        .gridSize(4)  // 4 threads
        .taskExecutor(taskExecutor())
        .build();
}

@Bean
public Partitioner partitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        
        // Divide by ID ranges
        long totalRecords = productRepository.count();
        long partitionSize = totalRecords / gridSize;
        
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", i * partitionSize + 1);
            ctx.putLong("maxId", (i == gridSize - 1) ? Long.MAX_VALUE : (i + 1) * partitionSize);
            partitions.put("partition" + i, ctx);
        }
        
        return partitions;
    };
}

@Bean
@StepScope
public JpaPagingItemReader<Product> partitionReader(
    @Value("#{stepExecutionContext['minId']}") Long minId,
    @Value("#{stepExecutionContext['maxId']}") Long maxId
) {
    return new JpaPagingItemReaderBuilder<Product>()
        .name("partitionReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT p FROM Product p WHERE p.id BETWEEN :minId AND :maxId")
        .parameterValues(Map.of("minId", minId, "maxId", maxId))
        .pageSize(100)
        .build();
}
```

---

## ขั้นตอนที่ 1485: Scheduled Batch Jobs

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BatchScheduler {
    
    private final JobLauncher jobLauncher;
    private final Job productImportJob;
    private final Job reportGenerationJob;
    
    // Daily at 2 AM
    @Scheduled(cron = "0 0 2 * * *")
    public void runDailyImport() {
        String importDir = "/data/imports/";
        
        try {
            File[] csvFiles = new File(importDir).listFiles((d, n) -> n.endsWith(".csv"));
            
            if (csvFiles == null || csvFiles.length == 0) {
                log.info("No CSV files to import");
                return;
            }
            
            for (File file : csvFiles) {
                JobParameters params = new JobParametersBuilder()
                    .addString("file.path", file.getAbsolutePath())
                    .addString("run.date", LocalDate.now().toString())
                    .addLong("timestamp", System.currentTimeMillis())
                    .toJobParameters();
                
                JobExecution execution = jobLauncher.run(productImportJob, params);
                log.info("Job {} finished with status {}", file.getName(), execution.getStatus());
            }
        } catch (Exception e) {
            log.error("Batch import failed", e);
        }
    }
    
    // Weekly report every Sunday at midnight
    @Scheduled(cron = "0 0 0 * * SUN")
    public void generateWeeklyReport() {
        try {
            JobParameters params = new JobParametersBuilder()
                .addString("report.period", "WEEKLY")
                .addString("week", String.valueOf(LocalDate.now().get(IsoFields.WEEK_OF_WEEK_BASED_YEAR)))
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();
            
            jobLauncher.run(reportGenerationJob, params);
        } catch (Exception e) {
            log.error("Report generation failed", e);
        }
    }
}
```

---

## ขั้นตอนที่ 1486-1520: REST API to Trigger Batch Jobs

```java
@RestController
@RequestMapping("/api/v1/admin/batch")
@PreAuthorize("hasRole('ADMIN')")
@RequiredArgsConstructor
public class BatchController {
    
    private final JobLauncher jobLauncher;
    private final JobExplorer jobExplorer;
    private final Job productImportJob;
    
    @PostMapping("/import")
    public ResponseEntity<BatchJobResponse> triggerImport(
        @RequestParam String filePath
    ) throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addString("file.path", filePath)
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        JobExecution execution = jobLauncher.run(productImportJob, params);
        
        return ResponseEntity.accepted().body(new BatchJobResponse(
            execution.getJobId(),
            execution.getStatus().toString(),
            execution.getStartTime()
        ));
    }
    
    @GetMapping("/jobs/{jobId}")
    public ResponseEntity<BatchJobStatusResponse> getJobStatus(@PathVariable Long jobId) {
        JobExecution execution = jobExplorer.getJobExecution(jobId);
        
        if (execution == null) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok(new BatchJobStatusResponse(
            execution.getJobId(),
            execution.getStatus().toString(),
            execution.getStartTime(),
            execution.getEndTime(),
            execution.getStepExecutions().stream()
                .map(s -> new StepStatus(s.getStepName(), s.getReadCount(), s.getWriteCount(), s.getSkipCount()))
                .toList()
        ));
    }
    
    @GetMapping("/jobs")
    public ResponseEntity<List<BatchJobStatusResponse>> getRecentJobs() {
        List<JobInstance> instances = jobExplorer.getJobInstances("productImportJob", 0, 10);
        
        return ResponseEntity.ok(instances.stream()
            .flatMap(i -> jobExplorer.getJobExecutions(i).stream())
            .sorted(Comparator.comparing(JobExecution::getCreateTime).reversed())
            .map(e -> new BatchJobStatusResponse(e.getJobId(), e.getStatus().toString(), 
                e.getStartTime(), e.getEndTime(), List.of()))
            .toList());
    }
}
```

---

*[← Part 46: Multi-Tenancy](./part-46-multitenancy.md) | [Part 48: Advanced Caching →](./part-48-advanced-caching.md)*

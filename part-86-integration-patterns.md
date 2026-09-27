# Part 86: Integration Patterns
## ขั้นตอนที่ 3041-3080

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้ Spring Integration framework สำหรับการเชื่อมต่อระบบต่างๆ ผ่าน message channels, endpoints, และ integration flows ระดับ enterprise

---

## ขั้นตอนที่ 3041: Spring Integration คืออะไร?

Spring Integration เป็น framework ที่ใช้สำหรับการเชื่อมต่อระบบต่างๆ โดยอิงจากแนวคิดของ Enterprise Integration Patterns (EIP) ซึ่งช่วยให้เราสร้าง messaging-based applications ได้อย่างมีโครงสร้าง

### เพิ่ม Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-integration</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-file</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-ftp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-sftp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-http</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-mail</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 3042: Message Channels พื้นฐาน

Message Channel คือท่อสำหรับส่ง messages ระหว่าง components ใน integration flow มีหลายประเภท

```java
// config/IntegrationChannelConfig.java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.channel.*;
import org.springframework.integration.dsl.MessageChannels;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.PollableChannel;

@Configuration
public class IntegrationChannelConfig {

    // Direct Channel - ส่ง message โดยตรง แบบ synchronous
    @Bean
    public MessageChannel directChannel() {
        return new DirectChannel();
    }

    // Queue Channel - เก็บ messages ไว้ใน queue สำหรับ async processing
    @Bean
    public PollableChannel queueChannel() {
        return new QueueChannel(100); // capacity 100 messages
    }

    // Publish-Subscribe Channel - ส่งไปหา subscribers ทุกคน
    @Bean
    public MessageChannel pubSubChannel() {
        return new PublishSubscribeChannel();
    }

    // Priority Channel - จัดลำดับตาม priority
    @Bean
    public PollableChannel priorityChannel() {
        return new PriorityChannel(50, (m1, m2) -> {
            Integer p1 = (Integer) m1.getHeaders().get("priority");
            Integer p2 = (Integer) m2.getHeaders().get("priority");
            if (p1 == null) p1 = 0;
            if (p2 == null) p2 = 0;
            return p2.compareTo(p1); // สูงกว่าได้ก่อน
        });
    }

    // Rendezvous Channel - ต้องมีทั้ง sender และ receiver พร้อมกัน
    @Bean
    public PollableChannel rendezvousChannel() {
        return new RendezvousChannel();
    }

    // ใช้ DSL สร้าง channel
    @Bean
    public MessageChannel orderInputChannel() {
        return MessageChannels.direct("orderInput").get();
    }

    @Bean
    public MessageChannel processedOrderChannel() {
        return MessageChannels.queue("processedOrders", 200).get();
    }
}
```

---

## ขั้นตอนที่ 3043: Message Endpoints

Endpoints คือจุดประมวลผล messages ใน integration flow มีหลายรูปแบบ

```java
// model/Order.java
package com.example.integration.model;

import lombok.Data;
import lombok.Builder;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@Builder
public class Order {
    private String orderId;
    private String customerId;
    private BigDecimal amount;
    private String status;
    private LocalDateTime createdAt;
    private String productCode;
    private Integer quantity;
}
```

```java
// service/OrderProcessingService.java
package com.example.integration.service;

import com.example.integration.model.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.integration.annotation.*;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Service
public class OrderProcessingService {

    // Service Activator - รับและประมวลผล message
    @ServiceActivator(inputChannel = "orderInputChannel")
    public Order processOrder(Order order) {
        log.info("Processing order: {}", order.getOrderId());
        
        // ตรวจสอบและอัปเดต order
        order.setStatus("PROCESSING");
        
        // จำลองการประมวลผล
        if (order.getAmount().compareTo(new BigDecimal("1000")) > 0) {
            order.setStatus("REQUIRES_APPROVAL");
        }
        
        return order;
    }

    // Transformer - แปลงรูปแบบ message
    @Transformer(inputChannel = "rawOrderChannel", outputChannel = "orderInputChannel")
    public Order transformRawOrder(String rawOrderData) {
        log.info("Transforming raw order: {}", rawOrderData);
        // แปลง raw string เป็น Order object
        String[] parts = rawOrderData.split(",");
        return Order.builder()
                .orderId(parts[0])
                .customerId(parts[1])
                .amount(new BigDecimal(parts[2]))
                .productCode(parts[3])
                .quantity(Integer.parseInt(parts[4]))
                .status("NEW")
                .createdAt(LocalDateTime.now())
                .build();
    }

    // Filter - กรอง messages
    @Filter(inputChannel = "orderInputChannel", outputChannel = "validOrderChannel",
            discardChannel = "invalidOrderChannel")
    public boolean filterValidOrders(Order order) {
        return order.getAmount().compareTo(BigDecimal.ZERO) > 0
                && order.getCustomerId() != null
                && !order.getCustomerId().isEmpty();
    }

    // Splitter - แยก message เป็นหลาย messages
    @Splitter(inputChannel = "bulkOrderChannel", outputChannel = "orderInputChannel")
    public List<Order> splitBulkOrder(List<Order> orders) {
        log.info("Splitting bulk order of {} items", orders.size());
        return orders; // แต่ละ order จะถูกส่งแยกกัน
    }

    // Router - กำหนดเส้นทาง message
    @Router(inputChannel = "orderRouter")
    public String routeOrder(@Payload Order order, @Header MessageHeaders headers) {
        if ("URGENT".equals(order.getStatus())) {
            return "urgentOrderChannel";
        } else if (order.getAmount().compareTo(new BigDecimal("10000")) > 0) {
            return "highValueOrderChannel";
        }
        return "standardOrderChannel";
    }
}
```

---

## ขั้นตอนที่ 3044: Integration Flow DSL

Spring Integration DSL ช่วยให้เขียน integration flows ได้ด้วย Java code แบบ fluent API

```java
// config/OrderIntegrationFlow.java
package com.example.integration.config;

import com.example.integration.model.Order;
import com.example.integration.service.OrderProcessingService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.*;
import org.springframework.integration.handler.LoggingHandler;
import org.springframework.integration.support.MessageBuilder;

import java.math.BigDecimal;
import java.util.concurrent.Executors;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class OrderIntegrationFlow {

    private final OrderProcessingService orderProcessingService;

    @Bean
    public IntegrationFlow orderProcessingFlow() {
        return IntegrationFlow
            .from("orderInputChannel")
            // Log ทุก message ที่เข้ามา
            .log(LoggingHandler.Level.INFO, "ORDER_FLOW", m -> "Received: " + m.getPayload())
            // กรองเฉพาะ order ที่ valid
            .filter(Order.class, order -> order.getAmount().compareTo(BigDecimal.ZERO) > 0)
            // แปลง/เพิ่ม header
            .enrichHeaders(h -> h
                .header("processedAt", System.currentTimeMillis())
                .headerExpression("orderPriority",
                    "payload.amount > 5000 ? 'HIGH' : 'NORMAL'"))
            // ประมวลผล order
            .handle(Order.class, (order, headers) -> {
                order.setStatus("PROCESSED");
                return order;
            })
            // Route ตามเงื่อนไข
            .<Order, String>route(
                order -> order.getAmount().compareTo(new BigDecimal("10000")) > 0
                    ? "highValue" : "standard",
                mapping -> mapping
                    .subFlowMapping("highValue", sf -> sf
                        .channel("highValueOrderChannel"))
                    .subFlowMapping("standard", sf -> sf
                        .channel("processedOrderChannel"))
            )
            .get();
    }

    // Async flow ด้วย executor channel
    @Bean
    public IntegrationFlow asyncOrderFlow() {
        return IntegrationFlow
            .from("asyncOrderInput")
            .channel(c -> c.executor(Executors.newFixedThreadPool(5)))
            .handle(Order.class, (order, headers) -> {
                log.info("Processing async order {} on thread: {}",
                    order.getOrderId(), Thread.currentThread().getName());
                // ทำงานใน thread pool แยก
                return order;
            })
            .channel("processedOrderChannel")
            .get();
    }

    // Scatter-Gather pattern - ส่งไปหลาย services แล้วรวมผล
    @Bean
    public IntegrationFlow scatterGatherFlow() {
        return IntegrationFlow
            .from("scatterGatherInput")
            .scatterGather(
                scatterer -> scatterer
                    .applySequence(true)
                    .recipientFlow(f -> f.handle(Order.class, (order, h) -> {
                        // ตรวจสอบ inventory
                        log.info("Checking inventory for {}", order.getOrderId());
                        return order;
                    }))
                    .recipientFlow(f -> f.handle(Order.class, (order, h) -> {
                        // ตรวจสอบ credit
                        log.info("Checking credit for {}", order.getOrderId());
                        return order;
                    })),
                gatherer -> gatherer
                    .releaseStrategy(g -> g.size() == 2)
                    .outputProcessor(group -> {
                        // รวมผลลัพธ์
                        log.info("Gathering results: {} messages", group.size());
                        return group.getOne();
                    })
            )
            .channel("scatterGatherOutput")
            .get();
    }
}
```

---

## ขั้นตอนที่ 3045: File Polling และ Processing

Spring Integration รองรับการ poll ไฟล์จาก directory และประมวลผลอัตโนมัติ

```java
// config/FileIntegrationConfig.java
package com.example.integration.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.*;
import org.springframework.integration.file.FileHeaders;
import org.springframework.integration.file.dsl.Files;
import org.springframework.integration.file.filters.*;
import org.springframework.integration.file.transformer.FileToStringTransformer;
import org.springframework.integration.scheduling.PollerMetadata;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.concurrent.TimeUnit;

@Slf4j
@Configuration
public class FileIntegrationConfig {

    @Value("${integration.file.input-dir:/tmp/integration/input}")
    private String inputDir;

    @Value("${integration.file.output-dir:/tmp/integration/output}")
    private String outputDir;

    @Value("${integration.file.archive-dir:/tmp/integration/archive}")
    private String archiveDir;

    // Default poller - ใช้เป็น default สำหรับทุก poller
    @Bean(name = PollerMetadata.DEFAULT_POLLER)
    public PollerMetadata defaultPoller() {
        return Pollers.fixedDelay(1000).get(); // poll ทุก 1 วินาที
    }

    // Flow สำหรับ poll และประมวลผลไฟล์ CSV
    @Bean
    public IntegrationFlow filePollingFlow() {
        return IntegrationFlow
            .from(Files.inboundAdapter(new File(inputDir))
                    .patternFilter("*.csv")
                    .preventDuplicates(true)
                    .useWatchService(true),
                e -> e.poller(Pollers.fixedDelay(2, TimeUnit.SECONDS)
                        .maxMessagesPerPoll(5)))
            .log(LoggingHandler.Level.INFO, "FILE_POLLER",
                m -> "Found file: " + m.getHeaders().get(FileHeaders.FILENAME))
            // แปลงไฟล์เป็น String
            .transform(new FileToStringTransformer())
            // ประมวลผลแต่ละบรรทัด
            .split(String.class, content -> content.split("\n"))
            .filter(String.class, line -> !line.trim().isEmpty() && !line.startsWith("#"))
            .transform(String.class, this::parseCsvLine)
            .aggregate(a -> a
                .correlationExpression("headers['file_name']")
                .releaseExpression("size() == 100 OR (size() > 0 AND !headers['sequenceSize'].equals(headers['sequenceNumber']))")
                .groupTimeout(30000L)) // timeout 30 วินาที
            .handle(this::processBatch)
            .get();
    }

    // Flow สำหรับเขียนผลลัพธ์ลงไฟล์
    @Bean
    public IntegrationFlow fileWritingFlow() {
        return IntegrationFlow
            .from("fileOutputChannel")
            .transform(Object::toString)
            .handle(Files.outboundAdapter(new File(outputDir))
                    .fileNameGenerator(message -> {
                        String timestamp = LocalDateTime.now()
                            .format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
                        return "output_" + timestamp + ".txt";
                    })
                    .appendNewLine(true))
            .get();
    }

    // Flow สำหรับย้ายไฟล์ที่ประมวลผลแล้วไปยัง archive
    @Bean
    public IntegrationFlow fileArchivingFlow() {
        return IntegrationFlow
            .from("archiveChannel")
            .handle(Files.outboundGateway(new File(archiveDir))
                    .deleteSourceFiles(true)
                    .preserveTimestamp(true))
            .get();
    }

    private Object parseCsvLine(String line) {
        String[] parts = line.split(",");
        return parts; // ส่งต่อเป็น array
    }

    private void processBatch(Object batch, Object headers) {
        log.info("Processing batch: {}", batch);
    }
}
```

---

## ขั้นตอนที่ 3046: FTP/SFTP Integration

Spring Integration รองรับการรับ/ส่งไฟล์ผ่าน FTP และ SFTP

```java
// config/FtpIntegrationConfig.java
package com.example.integration.config;

import com.jcraft.jsch.ChannelSftp;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.net.ftp.FTPFile;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.Pollers;
import org.springframework.integration.file.remote.session.CachingSessionFactory;
import org.springframework.integration.file.remote.session.SessionFactory;
import org.springframework.integration.ftp.dsl.Ftp;
import org.springframework.integration.ftp.session.DefaultFtpSessionFactory;
import org.springframework.integration.sftp.dsl.Sftp;
import org.springframework.integration.sftp.session.DefaultSftpSessionFactory;

import java.io.File;

@Slf4j
@Configuration
public class FtpIntegrationConfig {

    @Value("${ftp.host:localhost}")
    private String ftpHost;

    @Value("${ftp.port:21}")
    private int ftpPort;

    @Value("${ftp.username:ftpuser}")
    private String ftpUsername;

    @Value("${ftp.password:ftppass}")
    private String ftpPassword;

    @Value("${sftp.host:localhost}")
    private String sftpHost;

    @Value("${sftp.port:22}")
    private int sftpPort;

    @Value("${sftp.username:sftpuser}")
    private String sftpUsername;

    @Value("${sftp.private-key:~/.ssh/id_rsa}")
    private String sftpPrivateKey;

    // FTP Session Factory
    @Bean
    public SessionFactory<FTPFile> ftpSessionFactory() {
        DefaultFtpSessionFactory factory = new DefaultFtpSessionFactory();
        factory.setHost(ftpHost);
        factory.setPort(ftpPort);
        factory.setUsername(ftpUsername);
        factory.setPassword(ftpPassword);
        factory.setClientMode(0); // ACTIVE_LOCAL_DATA_CONNECTION_MODE
        return new CachingSessionFactory<>(factory, 10);
    }

    // SFTP Session Factory (password)
    @Bean
    public SessionFactory<ChannelSftp.LsEntry> sftpSessionFactory() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory(true);
        factory.setHost(sftpHost);
        factory.setPort(sftpPort);
        factory.setUser(sftpUsername);
        factory.setPassword("sftpPassword");
        factory.setAllowUnknownKeys(true);
        return new CachingSessionFactory<>(factory, 5);
    }

    // SFTP Session Factory (private key)
    @Bean
    public SessionFactory<ChannelSftp.LsEntry> sftpKeySessionFactory() {
        DefaultSftpSessionFactory factory = new DefaultSftpSessionFactory(true);
        factory.setHost(sftpHost);
        factory.setPort(sftpPort);
        factory.setUser(sftpUsername);
        factory.setPrivateKey(new org.springframework.core.io.FileSystemResource(sftpPrivateKey));
        factory.setAllowUnknownKeys(true);
        return new CachingSessionFactory<>(factory, 5);
    }

    // FTP Inbound - ดึงไฟล์จาก FTP server
    @Bean
    public IntegrationFlow ftpInboundFlow() {
        return IntegrationFlow
            .from(Ftp.inboundAdapter(ftpSessionFactory())
                    .remoteDirectory("/incoming")
                    .regexFilter(".*\\.csv$")
                    .localDirectory(new File("/tmp/ftp-local"))
                    .deleteRemoteFiles(false)
                    .preserveTimestamp(true),
                e -> e.poller(Pollers.fixedDelay(5000)))
            .log("FTP_INBOUND", m -> "Downloaded: " + m.getPayload())
            .channel("fileProcessingChannel")
            .get();
    }

    // FTP Outbound - อัปโหลดไฟล์ไป FTP server
    @Bean
    public IntegrationFlow ftpOutboundFlow() {
        return IntegrationFlow
            .from("ftpUploadChannel")
            .handle(Ftp.outboundAdapter(ftpSessionFactory())
                    .remoteDirectory("/outgoing")
                    .fileNameGenerator(message ->
                        "processed_" + message.getHeaders().get("originalFilename"))
                    .autoCreateDirectory(true))
            .get();
    }

    // SFTP Inbound - ดึงไฟล์จาก SFTP server แบบ streaming
    @Bean
    public IntegrationFlow sftpStreamingFlow() {
        return IntegrationFlow
            .from(Sftp.inboundStreamingAdapter(sftpSessionFactory())
                    .remoteDirectory("/secure/incoming")
                    .regexFilter(".*\\.(csv|xlsx)$"),
                e -> e.poller(Pollers.fixedDelay(10000)))
            .log("SFTP_STREAM", m -> "Streaming: " + m.getHeaders())
            // ประมวลผล stream โดยไม่ต้องเก็บไฟล์ลง local disk
            .handle(this::processSftpStream)
            .get();
    }

    // SFTP Outbound Gateway - ส่งคำสั่งไปยัง SFTP server
    @Bean
    public IntegrationFlow sftpOutboundGatewayFlow() {
        return IntegrationFlow
            .from("sftpCommandChannel")
            .handle(Sftp.outboundGateway(sftpSessionFactory(),
                    org.springframework.integration.file.remote.gateway.AbstractRemoteFileOutboundGateway.Command.LS,
                    "payload")
                    .options(
                        org.springframework.integration.file.remote.gateway.AbstractRemoteFileOutboundGateway.Option.NAME_ONLY,
                        org.springframework.integration.file.remote.gateway.AbstractRemoteFileOutboundGateway.Option.RECURSIVE
                    ))
            .channel("sftpListResultChannel")
            .get();
    }

    private Object processSftpStream(Object payload, Object headers) {
        log.info("Processing SFTP stream");
        return payload;
    }
}
```

---

## ขั้นตอนที่ 3047: HTTP Inbound/Outbound Gateways

Spring Integration รองรับการรับและส่ง HTTP requests ผ่าน gateways

```java
// config/HttpIntegrationConfig.java
package com.example.integration.config;

import com.example.integration.model.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.http.dsl.Http;
import org.springframework.integration.http.inbound.HttpRequestHandlingMessagingGateway;
import org.springframework.integration.http.inbound.RequestMapping;
import org.springframework.web.util.UriComponentsBuilder;

import java.util.Map;

@Slf4j
@Configuration
public class HttpIntegrationConfig {

    // HTTP Inbound Gateway - รับ HTTP requests และส่งเป็น messages
    @Bean
    public IntegrationFlow httpInboundFlow() {
        return IntegrationFlow
            .from(Http.inboundGateway("/api/integration/orders")
                    .requestMapping(m -> m.methods(HttpMethod.POST))
                    .requestPayloadType(Order.class)
                    .statusCodeExpression(
                        org.springframework.integration.expression.ExpressionUtils
                            .createStandardEvaluationContext()
                            .toString()
                    ))
            .log("HTTP_INBOUND", m -> "HTTP Request: " + m.getPayload())
            .channel("orderInputChannel")
            .get();
    }

    // HTTP Inbound Channel Adapter - รับ HTTP แต่ไม่ต้องการ response
    @Bean
    public IntegrationFlow httpInboundEventFlow() {
        return IntegrationFlow
            .from(Http.inboundChannelAdapter("/api/integration/events")
                    .requestMapping(m -> m
                        .methods(HttpMethod.POST)
                        .consumes("application/json"))
                    .requestPayloadType(Map.class))
            .channel("eventProcessingChannel")
            .get();
    }

    // HTTP Outbound Gateway - ส่ง HTTP request ไปยัง external service
    @Bean
    public IntegrationFlow httpOutboundFlow() {
        return IntegrationFlow
            .from("httpOutboundChannel")
            .enrichHeaders(h -> h
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer ${external.api.token}"))
            .handle(Http.outboundGateway("http://external-service/api/orders")
                    .httpMethod(HttpMethod.POST)
                    .expectedResponseType(String.class)
                    .requestFactory(new org.springframework.http.client
                        .SimpleClientHttpRequestFactory()))
            .log("HTTP_RESPONSE", m -> "Response: " + m.getPayload())
            .channel("httpResponseChannel")
            .get();
    }

    // HTTP Outbound Gateway แบบ dynamic URL
    @Bean
    public IntegrationFlow dynamicHttpOutboundFlow() {
        return IntegrationFlow
            .from("dynamicHttpChannel")
            .handle(Http.outboundGateway(message -> {
                    // สร้าง URL แบบ dynamic จาก message
                    String orderId = (String) message.getHeaders().get("orderId");
                    return UriComponentsBuilder
                        .fromHttpUrl("http://order-service/api/orders/{orderId}")
                        .buildAndExpand(orderId)
                        .toUri();
                })
                .httpMethod(HttpMethod.GET)
                .expectedResponseType(Order.class))
            .channel("orderResponseChannel")
            .get();
    }

    // REST Template-based outbound with retry
    @Bean
    public IntegrationFlow httpWithRetryFlow() {
        return IntegrationFlow
            .from("retryHttpChannel")
            .handle(Http.outboundGateway("http://external-api/data")
                    .httpMethod(HttpMethod.GET)
                    .expectedResponseType(String.class),
                e -> e.requestHandlerAdviceChain(
                    retryAdvice()
                ))
            .channel("retryResponseChannel")
            .get();
    }

    @Bean
    public org.springframework.integration.handler.advice.RequestHandlerRetryAdvice retryAdvice() {
        org.springframework.integration.handler.advice.RequestHandlerRetryAdvice advice =
            new org.springframework.integration.handler.advice.RequestHandlerRetryAdvice();
        org.springframework.retry.support.RetryTemplate retryTemplate =
            new org.springframework.retry.support.RetryTemplate();
        
        org.springframework.retry.backoff.ExponentialBackOffPolicy backOffPolicy =
            new org.springframework.retry.backoff.ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(1000);
        backOffPolicy.setMultiplier(2.0);
        backOffPolicy.setMaxInterval(30000);
        
        org.springframework.retry.policy.SimpleRetryPolicy retryPolicy =
            new org.springframework.retry.policy.SimpleRetryPolicy(3);
        
        retryTemplate.setBackOffPolicy(backOffPolicy);
        retryTemplate.setRetryPolicy(retryPolicy);
        advice.setRetryTemplate(retryTemplate);
        return advice;
    }
}
```

---

## ขั้นตอนที่ 3048: Message Transformers

Transformers ใช้สำหรับแปลงรูปแบบของ messages

```java
// transformer/OrderTransformer.java
package com.example.integration.transformer;

import com.example.integration.model.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.integration.annotation.Transformer;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@Component
public class OrderTransformer {

    // แปลง Order เป็น Map (JSON-like structure)
    @Transformer(inputChannel = "orderToMapChannel", outputChannel = "mapOutputChannel")
    public Map<String, Object> orderToMap(Order order) {
        Map<String, Object> map = new HashMap<>();
        map.put("id", order.getOrderId());
        map.put("customer", order.getCustomerId());
        map.put("amount", order.getAmount().toString());
        map.put("status", order.getStatus());
        map.put("timestamp", System.currentTimeMillis());
        return map;
    }

    // แปลง Map กลับเป็น Order
    @Transformer(inputChannel = "mapToOrderChannel", outputChannel = "orderOutputChannel")
    @SuppressWarnings("unchecked")
    public Order mapToOrder(Map<String, Object> map) {
        return Order.builder()
                .orderId((String) map.get("id"))
                .customerId((String) map.get("customer"))
                .amount(new BigDecimal(map.get("amount").toString()))
                .status((String) map.get("status"))
                .build();
    }

    // Content Enricher - เพิ่มข้อมูลให้ message
    @Transformer(inputChannel = "enrichOrderChannel", outputChannel = "enrichedOrderChannel")
    public Message<Order> enrichOrder(Order order) {
        // เพิ่ม metadata
        order.setStatus("ENRICHED");
        return MessageBuilder.withPayload(order)
                .setHeader("enrichedAt", System.currentTimeMillis())
                .setHeader("region", determineRegion(order.getCustomerId()))
                .setHeader("taxRate", calculateTaxRate(order))
                .build();
    }

    private String determineRegion(String customerId) {
        if (customerId.startsWith("TH")) return "THAILAND";
        if (customerId.startsWith("SG")) return "SINGAPORE";
        return "UNKNOWN";
    }

    private BigDecimal calculateTaxRate(Order order) {
        return new BigDecimal("0.07"); // VAT 7%
    }
}
```

---

## ขั้นตอนที่ 3049: Error Handling ใน Integration Flows

การจัดการ errors อย่างถูกต้องใน integration flows เป็นสิ่งสำคัญมาก

```java
// config/ErrorHandlingConfig.java
package com.example.integration.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.StandardIntegrationFlow;
import org.springframework.integration.handler.advice.*;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessagingException;
import org.springframework.retry.policy.SimpleRetryPolicy;
import org.springframework.retry.support.RetryTemplate;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@Configuration
public class ErrorHandlingConfig {

    // Error Channel สำหรับ global error handling
    @Bean
    public MessageChannel errorChannel() {
        return new DirectChannel();
    }

    // Global Error Handler
    @Bean
    public IntegrationFlow globalErrorFlow() {
        return IntegrationFlow
            .from("errorChannel")
            .handle(MessagingException.class, (exception, headers) -> {
                log.error("Integration error occurred: {}", exception.getMessage());
                Message<?> failedMessage = exception.getFailedMessage();
                if (failedMessage != null) {
                    log.error("Failed message: {}", failedMessage.getPayload());
                    log.error("Headers: {}", failedMessage.getHeaders());
                }
                
                // ส่งไปยัง dead letter channel
                return MessageBuilder
                    .withPayload(exception.getMessage())
                    .copyHeadersIfAbsent(failedMessage != null ?
                        failedMessage.getHeaders() : headers)
                    .setHeader("error.type", exception.getClass().getSimpleName())
                    .setHeader("error.timestamp", System.currentTimeMillis())
                    .build();
            })
            .channel("deadLetterChannel")
            .get();
    }

    // Dead Letter Channel Handler
    @Bean
    public IntegrationFlow deadLetterFlow() {
        return IntegrationFlow
            .from("deadLetterChannel")
            .log("DEAD_LETTER", m -> "Dead letter: " + m.getPayload())
            .handle(this::saveToDeadLetterStore)
            .get();
    }

    // Flow with Retry advice
    @Bean
    public IntegrationFlow flowWithRetry() {
        return IntegrationFlow
            .from("retryableChannel")
            .handle(this::unreliableOperation,
                e -> e.requestHandlerAdviceChain(retryAdvice()))
            .channel("successChannel")
            .get();
    }

    // Flow with Circuit Breaker
    @Bean
    public IntegrationFlow flowWithCircuitBreaker() {
        return IntegrationFlow
            .from("circuitBreakerChannel")
            .handle(this::externalServiceCall,
                e -> e.requestHandlerAdviceChain(circuitBreakerAdvice()))
            .channel("cbSuccessChannel")
            .get();
    }

    // Flow with Fallback
    @Bean
    public IntegrationFlow flowWithFallback() {
        return IntegrationFlow
            .from("fallbackChannel")
            .handle(this::primaryOperation,
                e -> e.requestHandlerAdviceChain(fallbackAdvice()))
            .channel("fallbackSuccessChannel")
            .get();
    }

    @Bean
    public RequestHandlerRetryAdvice retryAdvice() {
        RequestHandlerRetryAdvice advice = new RequestHandlerRetryAdvice();
        
        RetryTemplate retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(1000, 2, 10000)
            .retryOn(RuntimeException.class)
            .build();
        
        advice.setRetryTemplate(retryTemplate);
        
        // Recovery callback เมื่อ retry หมด
        advice.setRecoveryCallback(context -> {
            log.warn("Max retries exceeded. Sending to error channel.");
            Throwable lastThrowable = context.getLastThrowable();
            return MessageBuilder
                .withPayload("RETRY_EXHAUSTED: " + lastThrowable.getMessage())
                .build();
        });
        
        return advice;
    }

    @Bean
    public RequestHandlerCircuitBreakerAdvice circuitBreakerAdvice() {
        RequestHandlerCircuitBreakerAdvice advice = new RequestHandlerCircuitBreakerAdvice();
        advice.setThreshold(5); // เปิด circuit หลังจาก 5 failures
        advice.setHalfOpenAfter(30000L); // ลอง half-open หลัง 30 วินาที
        return advice;
    }

    @Bean
    public ExpressionEvaluatingRequestHandlerAdvice fallbackAdvice() {
        ExpressionEvaluatingRequestHandlerAdvice advice =
            new ExpressionEvaluatingRequestHandlerAdvice();
        advice.setOnFailureExpression(
            org.springframework.integration.expression.FunctionExpression.from(
                message -> {
                    log.warn("Primary operation failed, using fallback");
                    return "FALLBACK_VALUE";
                }
            )
        );
        advice.setReturnFailureExpressionResult(true);
        return advice;
    }

    private Object unreliableOperation(Object payload, Object headers) {
        // จำลอง operation ที่อาจล้มเหลว
        if (Math.random() < 0.7) {
            throw new RuntimeException("Simulated failure");
        }
        return payload;
    }

    private Object externalServiceCall(Object payload, Object headers) {
        // เรียก external service
        return payload;
    }

    private Object primaryOperation(Object payload, Object headers) {
        throw new RuntimeException("Primary operation failed");
    }

    private void saveToDeadLetterStore(Object payload, Object headers) {
        log.error("Saving to dead letter store: {}", payload);
        // บันทึกลงฐานข้อมูลหรือไฟล์
    }
}
```

---

## ขั้นตอนที่ 3050: Integration Aggregator และ Resequencer

Aggregator รวม messages หลายอัน, Resequencer เรียงลำดับ messages

```java
// config/AggregatorConfig.java
package com.example.integration.config;

import com.example.integration.model.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.Pollers;
import org.springframework.integration.store.MessageGroupStore;
import org.springframework.integration.store.SimpleMessageStore;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Configuration
public class AggregatorConfig {

    // Message Group Store สำหรับเก็บ messages ขณะรอ aggregate
    @Bean
    public MessageGroupStore messageGroupStore() {
        return new SimpleMessageStore(1000);
    }

    // Order Aggregator - รวม orders เป็น batch
    @Bean
    public IntegrationFlow orderAggregatorFlow() {
        return IntegrationFlow
            .from("orderSplitChannel")
            .aggregate(a -> a
                // กำหนดว่า messages ไหนอยู่กลุ่มเดียวกัน
                .correlationExpression("headers['batchId']")
                // เงื่อนไขการ release group
                .releaseExpression("size() == headers['batchSize']")
                // สิ่งที่ทำเมื่อ release
                .outputProcessor(group -> {
                    List<Order> orders = group.getMessages().stream()
                        .map(m -> (Order) m.getPayload())
                        .collect(Collectors.toList());
                    log.info("Aggregated {} orders", orders.size());
                    return orders;
                })
                // timeout ถ้าไม่ครบ
                .groupTimeout(60000L)
                .expireGroupsUponTimeout(true)
                .messageStore(messageGroupStore())
            )
            .channel("batchOrderChannel")
            .get();
    }

    // Resequencer - เรียงลำดับ messages ที่มาไม่ตรงลำดับ
    @Bean
    public IntegrationFlow resequencerFlow() {
        return IntegrationFlow
            .from("unorderedChannel")
            .resequence(r -> r
                .correlationExpression("headers['sessionId']")
                .releasePartialSequences(false) // รอให้ครบก่อน
                .messageStore(messageGroupStore())
            )
            .channel("orderedChannel")
            .get();
    }

    // Claim Check Pattern - เก็บ payload ขนาดใหญ่แยกต่างหาก
    @Bean
    public IntegrationFlow claimCheckFlow() {
        return IntegrationFlow
            .from("largePayloadChannel")
            // เก็บ payload ลง store และส่งแค่ claim check (key)
            .claimCheckIn(messageGroupStore())
            .channel("processingChannel")
            .get();
    }

    @Bean
    public IntegrationFlow claimCheckOutFlow() {
        return IntegrationFlow
            .from("retrieveChannel")
            // ดึง payload กลับมาจาก claim check
            .claimCheckOut(messageGroupStore())
            .channel("fullPayloadChannel")
            .get();
    }
}
```

---

## ขั้นตอนที่ 3051: Messaging Gateway

Messaging Gateway ทำหน้าที่เป็น interface ที่ clean สำหรับ integration

```java
// gateway/OrderGateway.java
package com.example.integration.gateway;

import com.example.integration.model.Order;
import org.springframework.integration.annotation.Gateway;
import org.springframework.integration.annotation.MessagingGateway;
import org.springframework.messaging.handler.annotation.Header;

import java.util.List;
import java.util.concurrent.Future;

@MessagingGateway(defaultRequestChannel = "orderGatewayChannel",
                  errorChannel = "gatewayErrorChannel")
public interface OrderGateway {

    // ส่ง order แบบ fire-and-forget
    @Gateway(requestChannel = "orderInputChannel")
    void submitOrder(Order order);

    // ส่ง order และรอผลลัพธ์
    @Gateway(requestChannel = "orderInputChannel",
             replyChannel = "orderResponseChannel",
             requestTimeout = 5000,
             replyTimeout = 10000)
    Order processOrder(Order order);

    // ส่ง order แบบ async
    @Gateway(requestChannel = "asyncOrderInput")
    Future<Order> processOrderAsync(Order order);

    // ส่ง bulk orders
    @Gateway(requestChannel = "bulkOrderChannel")
    void submitBulkOrders(List<Order> orders);

    // ส่ง order พร้อม header
    @Gateway(requestChannel = "priorityOrderChannel")
    void submitPriorityOrder(Order order,
        @Header("priority") int priority,
        @Header("correlationId") String correlationId);
}
```

```java
// controller/OrderIntegrationController.java
package com.example.integration.controller;

import com.example.integration.gateway.OrderGateway;
import com.example.integration.model.Order;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.concurrent.Future;

@Slf4j
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderIntegrationController {

    private final OrderGateway orderGateway;

    @PostMapping("/submit")
    public ResponseEntity<String> submitOrder(@RequestBody Order order) {
        orderGateway.submitOrder(order);
        return ResponseEntity.accepted().body("Order submitted");
    }

    @PostMapping("/process")
    public ResponseEntity<Order> processOrder(@RequestBody Order order) {
        Order processed = orderGateway.processOrder(order);
        return ResponseEntity.ok(processed);
    }

    @PostMapping("/bulk")
    public ResponseEntity<String> submitBulk(@RequestBody List<Order> orders) {
        orderGateway.submitBulkOrders(orders);
        return ResponseEntity.accepted().body("Bulk orders submitted: " + orders.size());
    }

    @PostMapping("/priority")
    public ResponseEntity<String> submitPriority(
            @RequestBody Order order,
            @RequestParam(defaultValue = "5") int priority) {
        String correlationId = java.util.UUID.randomUUID().toString();
        orderGateway.submitPriorityOrder(order, priority, correlationId);
        return ResponseEntity.accepted()
            .header("X-Correlation-Id", correlationId)
            .body("Priority order submitted");
    }
}
```

---

## ขั้นตอนที่ 3052: Testing Integration Flows

การทดสอบ integration flows ด้วย Spring Integration Test support

```java
// test/OrderIntegrationFlowTest.java
package com.example.integration;

import com.example.integration.model.Order;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.integration.test.context.SpringIntegrationTest;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.PollableChannel;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@SpringIntegrationTest
class OrderIntegrationFlowTest {

    @Autowired
    private MessageChannel orderInputChannel;

    @Autowired
    private PollableChannel processedOrderChannel;

    @Autowired
    private MessageChannel rawOrderChannel;

    @Test
    void shouldProcessOrderThroughFlow() {
        // สร้าง test order
        Order order = Order.builder()
            .orderId("TEST-001")
            .customerId("CUST-123")
            .amount(new BigDecimal("500"))
            .productCode("PROD-A")
            .quantity(2)
            .status("NEW")
            .createdAt(LocalDateTime.now())
            .build();

        // ส่ง message เข้า flow
        Message<Order> message = MessageBuilder.withPayload(order)
            .setHeader("correlationId", "test-corr-001")
            .build();
        orderInputChannel.send(message);

        // รอรับผลลัพธ์
        Message<?> result = ((QueueChannel) processedOrderChannel)
            .receive(5000); // timeout 5 วินาที

        assertThat(result).isNotNull();
        Order processedOrder = (Order) result.getPayload();
        assertThat(processedOrder.getStatus()).isEqualTo("PROCESSED");
    }

    @Test
    void shouldTransformRawOrderString() {
        // ส่ง raw CSV string
        String rawOrder = "ORD-002,CUST-456,1500.00,PROD-B,3";
        rawOrderChannel.send(MessageBuilder.withPayload(rawOrder).build());

        // ตรวจสอบว่า transform ถูกต้อง
        Message<?> result = ((QueueChannel) processedOrderChannel).receive(3000);
        assertThat(result).isNotNull();
        Order order = (Order) result.getPayload();
        assertThat(order.getOrderId()).isEqualTo("ORD-002");
        assertThat(order.getAmount()).isEqualByComparingTo("1500.00");
    }

    @Test
    void shouldFilterInvalidOrders() {
        // สร้าง invalid order (amount = 0)
        Order invalidOrder = Order.builder()
            .orderId("INVALID-001")
            .customerId("CUST-789")
            .amount(BigDecimal.ZERO)
            .build();

        orderInputChannel.send(MessageBuilder.withPayload(invalidOrder).build());

        // ไม่ควรผ่านไปยัง processed channel
        Message<?> result = ((QueueChannel) processedOrderChannel).receive(1000);
        assertThat(result).isNull();
    }
}
```

---

## ขั้นตอนที่ 3053: Integration Metrics และ Monitoring

ติดตาม metrics ของ integration flows

```java
// config/IntegrationMetricsConfig.java
package com.example.integration.config;

import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.channel.interceptor.WireTap;
import org.springframework.integration.config.EnableIntegrationManagement;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.monitor.IntegrationMBeanExporter;
import org.springframework.integration.support.management.*;

@Slf4j
@Configuration
@EnableIntegrationManagement(
    defaultLoggingEnabled = "true",
    countsEnabled = "true",
    statsEnabled = "true"
)
@RequiredArgsConstructor
public class IntegrationMetricsConfig {

    private final MeterRegistry meterRegistry;

    // Wire Tap - ดักจับ messages โดยไม่กระทบ flow หลัก
    @Bean
    public WireTap loggingWireTap() {
        return new WireTap("wireTapChannel");
    }

    // Wire Tap logging flow
    @Bean
    public IntegrationFlow wireTapFlow() {
        return IntegrationFlow
            .from("wireTapChannel")
            .handle(message -> {
                log.debug("Wire tap - Channel: {}, Payload type: {}",
                    message.getHeaders().get("channelName"),
                    message.getPayload().getClass().getSimpleName());
                
                // บันทึก metrics
                meterRegistry.counter("integration.messages.total",
                    "channel", String.valueOf(message.getHeaders().get("channelName")))
                    .increment();
            })
            .get();
    }

    // Custom Integration Management Configurer
    @Bean
    public IntegrationManagementConfigurer integrationManagementConfigurer() {
        IntegrationManagementConfigurer configurer = new IntegrationManagementConfigurer();
        configurer.setDefaultCountsEnabled(true);
        configurer.setDefaultStatsEnabled(true);
        return configurer;
    }
}
```

---

## Application Properties

```yaml
# application.yml
spring:
  integration:
    management:
      default-logging-enabled: true
      counts-enabled: "true"
      stats-enabled: "true"
    poller:
      fixed-delay: 1000

integration:
  file:
    input-dir: /tmp/integration/input
    output-dir: /tmp/integration/output
    archive-dir: /tmp/integration/archive

ftp:
  host: ftp.example.com
  port: 21
  username: ftpuser
  password: ${FTP_PASSWORD}

sftp:
  host: sftp.example.com
  port: 22
  username: sftpuser
  private-key: /home/app/.ssh/id_rsa

external:
  api:
    token: ${EXTERNAL_API_TOKEN}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,integrationgraph
  endpoint:
    health:
      show-details: always
```

---

## สรุป Part 86

ในส่วนนี้เราได้เรียนรู้:
- **Spring Integration Framework** - แนวคิด EIP และ message-driven architecture
- **Message Channels** - Direct, Queue, PubSub, Priority channels
- **Message Endpoints** - Service Activator, Transformer, Filter, Router, Splitter
- **Integration Flow DSL** - เขียน flows แบบ fluent Java API
- **File Integration** - Polling, reading, writing, archiving files
- **FTP/SFTP** - การรับส่งไฟล์ผ่าน FTP และ SFTP
- **HTTP Gateways** - รับและส่ง HTTP requests
- **Error Handling** - Retry, Circuit Breaker, Dead Letter
- **Aggregator/Resequencer** - รวมและเรียงลำดับ messages
- **Messaging Gateway** - Interface สำหรับ integration
- **Testing** - ทดสอบ integration flows

---

*[← Part 85: Advanced Caching](./part-85-advanced-caching.md) | [Part 87: Reactive Security →](./part-87-reactive-security.md)*

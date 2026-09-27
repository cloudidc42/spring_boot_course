# Part 80: Real-time Features
## ขั้นตอนที่ 2801-2840

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้การสร้างฟีเจอร์ real-time ด้วย WebSocket + STOMP, Server-Sent Events, Real-time chat, Live dashboard, Redis Pub/Sub และ Horizontal scaling

---

## ขั้นตอนที่ 2801: WebSocket และ STOMP Protocol

WebSocket ช่วยให้ client และ server สื่อสารแบบ two-way แบบ real-time ส่วน STOMP เป็น messaging protocol ที่ทำงานบน WebSocket ช่วยให้จัดการ destination/subscription ได้ง่ายขึ้น

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.webjars</groupId>
        <artifactId>sockjs-client</artifactId>
        <version>1.5.1</version>
    </dependency>
    <dependency>
        <groupId>org.webjars</groupId>
        <artifactId>stomp-websocket</artifactId>
        <version>2.3.4</version>
    </dependency>
</dependencies>
```

---

## ขั้นตอนที่ 2802: WebSocket Configuration

```java
// config/WebSocketConfig.java
package com.example.realtime.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.*;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Prefix สำหรับ subscribe (client listen)
        registry.enableSimpleBroker(
            "/topic",   // broadcast ไปทุก subscriber
            "/queue"    // ส่งไปยัง user เฉพาะ
        );

        // Prefix สำหรับ application messages (client send)
        registry.setApplicationDestinationPrefixes("/app");

        // Prefix สำหรับ user-specific messages
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS()  // Fallback สำหรับ browser ที่ไม่รองรับ WebSocket
                .setHeartbeatTime(25000)
                .setDisconnectDelay(5000);
    }

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registry) {
        registry.setMessageSizeLimit(64 * 1024);         // 64KB per message
        registry.setSendTimeLimit(20 * 1000);             // 20 วินาที timeout
        registry.setSendBufferSizeLimit(512 * 1024);      // 512KB buffer
        registry.setTimeToFirstMessage(30 * 1000);        // 30 วินาที timeout แรก
    }
}
```

---

## ขั้นตอนที่ 2803: Real-time Chat

```java
// model/ChatMessage.java
package com.example.realtime.model;

import lombok.Data;
import java.time.LocalDateTime;

@Data
public class ChatMessage {
    private String id;
    private String roomId;
    private String senderId;
    private String senderName;
    private String content;
    private MessageType type;
    private LocalDateTime timestamp;

    public enum MessageType {
        CHAT,       // ข้อความปกติ
        JOIN,       // เข้าร่วมห้อง
        LEAVE,      // ออกจากห้อง
        TYPING,     // กำลังพิมพ์
        SYSTEM      // ข้อความจากระบบ
    }
}
```

```java
// controller/ChatController.java
package com.example.realtime.controller;

import com.example.realtime.model.ChatMessage;
import com.example.realtime.service.ChatService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

import java.security.Principal;

@Slf4j
@Controller
@RequiredArgsConstructor
public class ChatController {

    private final SimpMessagingTemplate messagingTemplate;
    private final ChatService chatService;

    // Client ส่งข้อความมายัง /app/chat/{roomId}
    @MessageMapping("/chat/{roomId}")
    public void sendMessage(@DestinationVariable String roomId,
                             ChatMessage message,
                             Principal principal) {
        message.setSenderId(principal.getName());
        message.setType(ChatMessage.MessageType.CHAT);
        message.setTimestamp(java.time.LocalDateTime.now());

        // บันทึกลง database
        chatService.saveMessage(message);

        // Broadcast ไปยังทุกคนในห้อง
        messagingTemplate.convertAndSend(
            "/topic/chat/" + roomId, message
        );

        log.debug("Message sent to room {}: {}", roomId, message.getContent());
    }

    // Client เข้าร่วมห้อง
    @MessageMapping("/chat/{roomId}/join")
    public void joinRoom(@DestinationVariable String roomId,
                          SimpMessageHeaderAccessor headerAccessor,
                          Principal principal) {
        String username = principal.getName();

        // เพิ่ม user ลใน room session
        headerAccessor.getSessionAttributes().put("roomId", roomId);
        headerAccessor.getSessionAttributes().put("username", username);

        chatService.addUserToRoom(roomId, username);

        // แจ้งคนอื่นในห้อง
        ChatMessage joinMessage = new ChatMessage();
        joinMessage.setType(ChatMessage.MessageType.JOIN);
        joinMessage.setSenderId(username);
        joinMessage.setSenderName(username);
        joinMessage.setContent(username + " เข้าร่วมห้องสนทนา");
        joinMessage.setTimestamp(java.time.LocalDateTime.now());

        messagingTemplate.convertAndSend("/topic/chat/" + roomId, joinMessage);

        // ส่ง presence list ไปยัง user ที่เพิ่งเข้า
        var users = chatService.getUsersInRoom(roomId);
        messagingTemplate.convertAndSendToUser(
            username, "/queue/chat/" + roomId + "/users", users
        );
    }

    // Client กำลังพิมพ์
    @MessageMapping("/chat/{roomId}/typing")
    public void typing(@DestinationVariable String roomId,
                        Principal principal) {
        ChatMessage typingMessage = new ChatMessage();
        typingMessage.setType(ChatMessage.MessageType.TYPING);
        typingMessage.setSenderId(principal.getName());
        typingMessage.setSenderName(principal.getName());
        typingMessage.setTimestamp(java.time.LocalDateTime.now());

        messagingTemplate.convertAndSend(
            "/topic/chat/" + roomId + "/typing", typingMessage
        );
    }

    // ส่งข้อความส่วนตัว
    @MessageMapping("/private/{targetUser}")
    public void sendPrivateMessage(@DestinationVariable String targetUser,
                                    ChatMessage message,
                                    Principal principal) {
        message.setSenderId(principal.getName());
        message.setTimestamp(java.time.LocalDateTime.now());

        // ส่งไปยัง target user
        messagingTemplate.convertAndSendToUser(
            targetUser, "/queue/private", message
        );

        // ส่ง copy ไปยัง sender เองด้วย
        messagingTemplate.convertAndSendToUser(
            principal.getName(), "/queue/private", message
        );
    }
}
```

---

## ขั้นตอนที่ 2804: Server-Sent Events (SSE)

SSE เหมาะสำหรับ one-way push จาก server ไป client เช่น live feed, notifications

```java
// controller/SseController.java
package com.example.realtime.controller;

import com.example.realtime.service.SseNotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.concurrent.TimeUnit;

@Slf4j
@RestController
@RequestMapping("/api/v1/sse")
@RequiredArgsConstructor
public class SseController {

    private final SseNotificationService sseService;

    // Client เชื่อมต่อมาที่ SSE endpoint
    @GetMapping(value = "/notifications", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribeNotifications(
            @AuthenticationPrincipal UserDetails user) {

        // Timeout 30 นาที (client จะ reconnect อัตโนมัติ)
        SseEmitter emitter = new SseEmitter(TimeUnit.MINUTES.toMillis(30));

        String userId = user.getUsername();
        sseService.registerEmitter(userId, emitter);

        // ส่ง initial connection event
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data("Connected successfully")
            );
        } catch (IOException e) {
            sseService.removeEmitter(userId, emitter);
        }

        // Cleanup เมื่อ client disconnect
        emitter.onCompletion(() -> {
            sseService.removeEmitter(userId, emitter);
            log.debug("SSE connection completed for user: {}", userId);
        });

        emitter.onTimeout(() -> {
            sseService.removeEmitter(userId, emitter);
            log.debug("SSE connection timed out for user: {}", userId);
        });

        emitter.onError(e -> {
            sseService.removeEmitter(userId, emitter);
            log.debug("SSE connection error for user {}: {}", userId, e.getMessage());
        });

        return emitter;
    }

    // SSE สำหรับ live metrics (ไม่ต้อง auth)
    @GetMapping(value = "/metrics/live", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter liveMetrics() {
        SseEmitter emitter = new SseEmitter(TimeUnit.MINUTES.toMillis(5));
        sseService.registerMetricsEmitter(emitter);
        return emitter;
    }
}
```

```java
// service/SseNotificationService.java
package com.example.realtime.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

@Slf4j
@Service
public class SseNotificationService {

    // user ID -> list of emitters (user อาจมีหลาย tab/device)
    private final Map<String, List<SseEmitter>> userEmitters = new java.util.concurrent.ConcurrentHashMap<>();
    private final List<SseEmitter> metricsEmitters = new CopyOnWriteArrayList<>();

    public void registerEmitter(String userId, SseEmitter emitter) {
        userEmitters.computeIfAbsent(userId, k -> new CopyOnWriteArrayList<>()).add(emitter);
        log.debug("Registered SSE emitter for user: {}", userId);
    }

    public void removeEmitter(String userId, SseEmitter emitter) {
        List<SseEmitter> emitters = userEmitters.get(userId);
        if (emitters != null) {
            emitters.remove(emitter);
            if (emitters.isEmpty()) {
                userEmitters.remove(userId);
            }
        }
    }

    public void registerMetricsEmitter(SseEmitter emitter) {
        metricsEmitters.add(emitter);
        emitter.onCompletion(() -> metricsEmitters.remove(emitter));
        emitter.onTimeout(() -> metricsEmitters.remove(emitter));
    }

    // ส่ง notification ไปยัง user
    public void sendToUser(String userId, String eventName, Object data) {
        List<SseEmitter> emitters = userEmitters.getOrDefault(userId, Collections.emptyList());
        List<SseEmitter> deadEmitters = new ArrayList<>();

        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                    .name(eventName)
                    .data(data)
                    .id(UUID.randomUUID().toString())
                );
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }

        deadEmitters.forEach(e -> removeEmitter(userId, e));
    }

    // Broadcast metrics ไปยังทุก emitter
    public void broadcastMetrics(Object metricsData) {
        List<SseEmitter> deadEmitters = new ArrayList<>();

        for (SseEmitter emitter : metricsEmitters) {
            try {
                emitter.send(SseEmitter.event()
                    .name("metrics")
                    .data(metricsData)
                    .id(String.valueOf(System.currentTimeMillis()))
                );
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }

        metricsEmitters.removeAll(deadEmitters);
    }

    // Heartbeat เพื่อ keep connection alive
    public void sendHeartbeat() {
        String heartbeat = LocalDateTime.now().toString();
        userEmitters.forEach((userId, emitters) -> {
            emitters.forEach(emitter -> {
                try {
                    emitter.send(SseEmitter.event()
                        .name("heartbeat")
                        .data(heartbeat)
                        .comment("keep-alive")
                    );
                } catch (IOException e) {
                    // จะถูก clean up ในรอบถัดไป
                }
            });
        });
    }
}
```

---

## ขั้นตอนที่ 2805: Live Dashboard ด้วย Metrics Streaming

```java
// service/MetricsStreamingService.java
package com.example.realtime.service;

import io.micrometer.core.instrument.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class MetricsStreamingService {

    private final MeterRegistry meterRegistry;
    private final SseNotificationService sseService;

    // ส่ง metrics ทุก 5 วินาที
    @Scheduled(fixedDelay = 5000)
    public void streamMetrics() {
        Map<String, Object> metrics = collectMetrics();
        sseService.broadcastMetrics(metrics);
    }

    private Map<String, Object> collectMetrics() {
        Map<String, Object> metrics = new HashMap<>();

        // JVM Memory
        metrics.put("jvm.memory.used",
            getGaugeValue("jvm.memory.used", "area", "heap"));
        metrics.put("jvm.memory.max",
            getGaugeValue("jvm.memory.max", "area", "heap"));

        // HTTP requests
        metrics.put("http.requests.total",
            getCounterValue("http.server.requests"));

        // Active connections
        metrics.put("active.connections",
            getGaugeValue("tomcat.threads.busy", null, null));

        // Custom business metrics
        metrics.put("orders.today", getCounterValue("orders.created.today"));
        metrics.put("revenue.today", getCounterValue("revenue.today"));
        metrics.put("active.users", getGaugeValue("active.users", null, null));

        metrics.put("timestamp", System.currentTimeMillis());

        return metrics;
    }

    private double getGaugeValue(String name, String tagKey, String tagValue) {
        try {
            Meter meter = tagKey != null
                ? meterRegistry.find(name).tag(tagKey, tagValue).meter()
                : meterRegistry.find(name).meter();
            if (meter instanceof Gauge gauge) return gauge.value();
            return 0.0;
        } catch (Exception e) {
            return 0.0;
        }
    }

    private double getCounterValue(String name) {
        try {
            Counter counter = meterRegistry.find(name).counter();
            return counter != null ? counter.count() : 0.0;
        } catch (Exception e) {
            return 0.0;
        }
    }
}
```

---

## ขั้นตอนที่ 2806: Redis Pub/Sub สำหรับ Notifications

Redis Pub/Sub ช่วย broadcast message ข้าม server instances ได้

```java
// config/RedisConfig.java
package com.example.realtime.config;

import com.example.realtime.service.RedisNotificationSubscriber;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.*;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    public static final String NOTIFICATION_CHANNEL = "notifications";
    public static final String CHAT_CHANNEL_PREFIX = "chat:";
    public static final String PRESENCE_CHANNEL = "presence";

    @Bean
    RedisMessageListenerContainer redisContainer(
            RedisConnectionFactory factory,
            MessageListenerAdapter notificationListener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);

        // Subscribe ไปยัง notifications channel
        container.addMessageListener(notificationListener,
            new PatternTopic(NOTIFICATION_CHANNEL));

        // Subscribe ไปยัง chat channels ทั้งหมด
        container.addMessageListener(notificationListener,
            new PatternTopic(CHAT_CHANNEL_PREFIX + "*"));

        return container;
    }

    @Bean
    MessageListenerAdapter notificationListener(RedisNotificationSubscriber subscriber) {
        return new MessageListenerAdapter(subscriber, "handleMessage");
    }

    @Bean
    RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        return template;
    }
}
```

```java
// service/RedisNotificationSubscriber.java
package com.example.realtime.service;

import com.example.realtime.model.NotificationMessage;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class RedisNotificationSubscriber {

    private final SimpMessagingTemplate messagingTemplate;
    private final SseNotificationService sseService;
    private final ObjectMapper objectMapper;

    // รับ message จาก Redis และกระจายไปยัง WebSocket/SSE clients
    public void handleMessage(String messageJson, String channel) {
        try {
            NotificationMessage notification = objectMapper.readValue(
                messageJson, NotificationMessage.class
            );

            log.debug("Received from Redis channel {}: {}", channel, notification);

            if (channel.startsWith(RedisConfig.CHAT_CHANNEL_PREFIX)) {
                // Forward ไปยัง WebSocket chat topic
                String roomId = channel.replace(RedisConfig.CHAT_CHANNEL_PREFIX, "");
                messagingTemplate.convertAndSend(
                    "/topic/chat/" + roomId, notification
                );
            } else if (channel.equals(RedisConfig.NOTIFICATION_CHANNEL)) {
                // ส่งไปยัง user ที่ระบุ
                if (notification.getUserId() != null) {
                    // WebSocket
                    messagingTemplate.convertAndSendToUser(
                        notification.getUserId(),
                        "/queue/notifications",
                        notification
                    );
                    // SSE
                    sseService.sendToUser(
                        notification.getUserId(),
                        "notification",
                        notification
                    );
                }
            }
        } catch (Exception e) {
            log.error("Failed to process Redis message: {}", e.getMessage());
        }
    }
}
```

```java
// service/RedisPublishService.java
package com.example.realtime.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class RedisPublishService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ObjectMapper objectMapper;

    // Publish notification ไปยัง Redis (สำหรับ broadcast ข้าม instances)
    public void publishNotification(String userId, Object data) {
        try {
            var message = new com.example.realtime.model.NotificationMessage();
            message.setUserId(userId);
            message.setData(data);

            redisTemplate.convertAndSend(
                RedisConfig.NOTIFICATION_CHANNEL,
                objectMapper.writeValueAsString(message)
            );
        } catch (Exception e) {
            log.error("Failed to publish notification: {}", e.getMessage());
        }
    }

    // Publish chat message ไปยัง channel เฉพาะ
    public void publishChatMessage(String roomId, Object message) {
        try {
            redisTemplate.convertAndSend(
                RedisConfig.CHAT_CHANNEL_PREFIX + roomId,
                objectMapper.writeValueAsString(message)
            );
        } catch (Exception e) {
            log.error("Failed to publish chat message: {}", e.getMessage());
        }
    }
}
```

---

## ขั้นตอนที่ 2807: WebSocket Security

```java
// config/WebSocketSecurityConfig.java
package com.example.realtime.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.handler.invocation.HandlerMethodArgumentResolver;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.security.authorization.AuthorizationManager;
import org.springframework.security.config.annotation.web.socket.EnableWebSocketSecurity;
import org.springframework.security.messaging.access.intercept.MessageMatcherDelegatingAuthorizationManager;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

import java.util.List;

@Configuration
@EnableWebSocketSecurity
public class WebSocketSecurityConfig implements WebSocketMessageBrokerConfigurer {

    @Bean
    AuthorizationManager<Message<?>> messageAuthorizationManager(
            MessageMatcherDelegatingAuthorizationManager.Builder messages) {

        return messages
            // Authenticate ทุก message ที่ส่งผ่าน /app
            .simpDestMatchers("/app/**").authenticated()
            // Subscribe ไปยัง user-specific queue ต้อง authenticated
            .simpSubscribeDestMatchers("/user/**").authenticated()
            // Allow subscribe ไปยัง public topics
            .simpSubscribeDestMatchers("/topic/public/**").permitAll()
            .simpSubscribeDestMatchers("/topic/**").authenticated()
            // ทุกอย่างอื่น deny
            .anyMessage().denyAll()
            .build();
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        // เพิ่ม JWT authentication interceptor
        registration.interceptors(new JwtChannelInterceptor());
    }
}
```

```java
// security/JwtChannelInterceptor.java
package com.example.realtime.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.*;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

import java.util.List;

@Slf4j
public class JwtChannelInterceptor implements ChannelInterceptor {

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = MessageHeaderAccessor
            .getAccessor(message, StompHeaderAccessor.class);

        if (accessor != null && StompCommand.CONNECT.equals(accessor.getCommand())) {
            String token = accessor.getFirstNativeHeader("Authorization");

            if (token != null && token.startsWith("Bearer ")) {
                String jwt = token.substring(7);
                try {
                    // Validate JWT และดึง user info
                    String userId = validateJwtAndGetUserId(jwt);
                    var auth = new UsernamePasswordAuthenticationToken(
                        userId, null,
                        List.of(new SimpleGrantedAuthority("ROLE_USER"))
                    );
                    accessor.setUser(auth);
                    log.debug("WebSocket authenticated: {}", userId);
                } catch (Exception e) {
                    log.warn("WebSocket JWT validation failed: {}", e.getMessage());
                    return null;  // Reject connection
                }
            }
        }

        return message;
    }

    private String validateJwtAndGetUserId(String jwt) {
        // Validate JWT token และ return user ID
        // ใช้ JwtService ที่มีอยู่แล้ว
        return "user-placeholder"; // placeholder
    }
}
```

---

## ขั้นตอนที่ 2808: Horizontal Scaling ด้วย Redis Message Broker

เมื่อ scale เป็นหลาย instance ต้องใช้ external message broker เพื่อ relay message

```java
// config/WebSocketScaleConfig.java
package com.example.realtime.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@Profile("production")  // ใช้เฉพาะใน production
@EnableWebSocketMessageBroker
public class WebSocketScaleConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // ใช้ STOMP relay ไปยัง RabbitMQ (external broker)
        config.enableStompBrokerRelay("/topic", "/queue")
            .setRelayHost("rabbitmq-host")
            .setRelayPort(61613)
            .setClientLogin("guest")
            .setClientPasscode("guest")
            .setSystemLogin("guest")
            .setSystemPasscode("guest")
            .setSystemHeartbeatSendInterval(10000)
            .setSystemHeartbeatReceiveInterval(10000);

        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS();
    }
}
```

---

## ขั้นตอนที่ 2809: Presence System

ระบบ presence แสดงว่า user ใดกำลัง online อยู่

```java
// service/PresenceService.java
package com.example.realtime.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Set;

@Slf4j
@Service
@RequiredArgsConstructor
public class PresenceService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String ONLINE_USERS_KEY = "online:users";
    private static final String USER_LAST_SEEN_PREFIX = "user:lastseen:";

    // Mark user เป็น online
    public void userConnected(String userId) {
        redisTemplate.opsForSet().add(ONLINE_USERS_KEY, userId);
        updateLastSeen(userId);
        log.debug("User connected: {}", userId);
    }

    // Mark user เป็น offline
    public void userDisconnected(String userId) {
        redisTemplate.opsForSet().remove(ONLINE_USERS_KEY, userId);
        updateLastSeen(userId);
        log.debug("User disconnected: {}", userId);
    }

    // ตรวจสอบว่า user online หรือไม่
    public boolean isOnline(String userId) {
        return Boolean.TRUE.equals(
            redisTemplate.opsForSet().isMember(ONLINE_USERS_KEY, userId)
        );
    }

    // ดึง list ของ online users
    @SuppressWarnings("unchecked")
    public Set<String> getOnlineUsers() {
        return (Set<String>) (Set<?>) redisTemplate.opsForSet().members(ONLINE_USERS_KEY);
    }

    // ดึง online users ใน room
    public Set<String> getOnlineUsersInRoom(String roomId, Set<String> roomMembers) {
        Set<String> onlineUsers = getOnlineUsers();
        if (onlineUsers == null) return Set.of();
        onlineUsers.retainAll(roomMembers);
        return onlineUsers;
    }

    // อัปเดต last seen timestamp
    public void updateLastSeen(String userId) {
        redisTemplate.opsForValue().set(
            USER_LAST_SEEN_PREFIX + userId,
            System.currentTimeMillis(),
            Duration.ofDays(30)
        );
    }

    // ดึง last seen time
    public Long getLastSeen(String userId) {
        Object value = redisTemplate.opsForValue().get(USER_LAST_SEEN_PREFIX + userId);
        return value instanceof Long l ? l : null;
    }

    // ดึงจำนวน online users
    public long getOnlineCount() {
        Long count = redisTemplate.opsForSet().size(ONLINE_USERS_KEY);
        return count != null ? count : 0;
    }
}
```

```java
// listener/WebSocketEventListener.java
package com.example.realtime.listener;

import com.example.realtime.service.PresenceService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.*;

import java.util.Map;

@Slf4j
@Component
@RequiredArgsConstructor
public class WebSocketEventListener {

    private final PresenceService presenceService;
    private final SimpMessagingTemplate messagingTemplate;

    // เมื่อ client เชื่อมต่อ
    @EventListener
    public void handleWebSocketConnect(SessionConnectedEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String userId = getUserId(accessor);
        if (userId != null) {
            presenceService.userConnected(userId);
            broadcastPresenceChange(userId, "ONLINE");
        }
    }

    // เมื่อ client disconnect
    @EventListener
    public void handleWebSocketDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String userId = getUserId(accessor);
        if (userId != null) {
            presenceService.userDisconnected(userId);
            broadcastPresenceChange(userId, "OFFLINE");

            // ส่ง leave message ถ้าอยู่ใน chat room
            String roomId = getRoomId(accessor);
            if (roomId != null) {
                var leaveMessage = Map.of(
                    "type", "LEAVE",
                    "userId", userId,
                    "roomId", roomId
                );
                messagingTemplate.convertAndSend("/topic/chat/" + roomId, leaveMessage);
            }
        }
    }

    private void broadcastPresenceChange(String userId, String status) {
        var presenceEvent = Map.of("userId", userId, "status", status);
        messagingTemplate.convertAndSend("/topic/presence", presenceEvent);
    }

    private String getUserId(StompHeaderAccessor accessor) {
        if (accessor.getUser() != null) {
            return accessor.getUser().getName();
        }
        return null;
    }

    private String getRoomId(StompHeaderAccessor accessor) {
        Map<String, Object> attrs = accessor.getSessionAttributes();
        if (attrs != null) {
            return (String) attrs.get("roomId");
        }
        return null;
    }
}
```

---

## ขั้นตอนที่ 2810: Client-side JavaScript

```javascript
// static/js/websocket-client.js

class RealtimeClient {
    constructor(serverUrl, token) {
        this.serverUrl = serverUrl;
        this.token = token;
        this.stompClient = null;
        this.subscriptions = new Map();
        this.reconnectDelay = 5000;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 10;
    }

    connect(onConnected, onError) {
        const socket = new SockJS(this.serverUrl + '/ws');
        this.stompClient = Stomp.over(socket);

        // Disable debug logging
        this.stompClient.debug = null;

        const headers = {
            Authorization: 'Bearer ' + this.token
        };

        this.stompClient.connect(headers, (frame) => {
            console.log('Connected:', frame);
            this.reconnectAttempts = 0;
            if (onConnected) onConnected(frame);
        }, (error) => {
            console.error('WebSocket error:', error);
            if (this.reconnectAttempts < this.maxReconnectAttempts) {
                console.log(`Reconnecting in ${this.reconnectDelay}ms...`);
                setTimeout(() => {
                    this.reconnectAttempts++;
                    this.connect(onConnected, onError);
                }, this.reconnectDelay * Math.pow(1.5, this.reconnectAttempts));
            } else {
                if (onError) onError(error);
            }
        });
    }

    // Subscribe ไปยัง topic
    subscribe(destination, callback) {
        if (!this.stompClient || !this.stompClient.connected) {
            throw new Error('Not connected');
        }
        const subscription = this.stompClient.subscribe(destination, (message) => {
            const body = JSON.parse(message.body);
            callback(body);
        });
        this.subscriptions.set(destination, subscription);
        return subscription;
    }

    // ส่ง message
    send(destination, body, headers = {}) {
        if (!this.stompClient || !this.stompClient.connected) {
            throw new Error('Not connected');
        }
        this.stompClient.send(destination, headers, JSON.stringify(body));
    }

    // Unsubscribe
    unsubscribe(destination) {
        const subscription = this.subscriptions.get(destination);
        if (subscription) {
            subscription.unsubscribe();
            this.subscriptions.delete(destination);
        }
    }

    disconnect() {
        if (this.stompClient) {
            this.stompClient.disconnect(() => {
                console.log('Disconnected');
            });
        }
    }
}

// ตัวอย่างการใช้งาน
const client = new RealtimeClient('http://localhost:8080', 'your-jwt-token');

client.connect(
    (frame) => {
        // Subscribe รับ notifications
        client.subscribe('/user/queue/notifications', (notification) => {
            showNotification(notification);
        });

        // Subscribe รับ chat messages ใน room
        client.subscribe('/topic/chat/room-123', (message) => {
            displayMessage(message);
        });

        // เข้าร่วม chat room
        client.send('/app/chat/room-123/join', {});
    },
    (error) => console.error('Connection failed:', error)
);

// ส่งข้อความ
function sendMessage(roomId, content) {
    client.send('/app/chat/' + roomId, { content });
}

// SSE Client
function connectSSE(token) {
    const eventSource = new EventSource(
        '/api/v1/sse/notifications',
        { headers: { Authorization: 'Bearer ' + token } }
    );

    eventSource.addEventListener('connected', (e) => {
        console.log('SSE connected:', e.data);
    });

    eventSource.addEventListener('notification', (e) => {
        const data = JSON.parse(e.data);
        showNotification(data);
    });

    eventSource.addEventListener('metrics', (e) => {
        const metrics = JSON.parse(e.data);
        updateDashboard(metrics);
    });

    eventSource.onerror = (e) => {
        console.error('SSE error, will auto-reconnect');
    };

    return eventSource;
}
```

---

## ขั้นตอนที่ 2811: ทดสอบ WebSocket

```java
// test/WebSocketIntegrationTest.java
package com.example.realtime.controller;

import com.example.realtime.model.ChatMessage;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.messaging.simp.stomp.*;
import org.springframework.web.socket.WebSocketHttpHeaders;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.messaging.WebSocketStompClient;

import java.lang.reflect.Type;
import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class WebSocketIntegrationTest {

    @LocalServerPort
    private int port;

    private WebSocketStompClient stompClient;

    @BeforeEach
    void setup() {
        stompClient = new WebSocketStompClient(new StandardWebSocketClient());
        stompClient.setMessageConverter(new MappingJackson2MessageConverter());
    }

    @Test
    void shouldReceiveChatMessage() throws Exception {
        BlockingQueue<ChatMessage> receivedMessages = new LinkedBlockingQueue<>();

        StompSession session = stompClient.connect(
            "ws://localhost:" + port + "/ws",
            new WebSocketHttpHeaders(),
            new StompSessionHandlerAdapter() {}
        ).get(5, TimeUnit.SECONDS);

        // Subscribe ไปยัง chat room
        session.subscribe("/topic/chat/room-test",
            new StompFrameHandler() {
                @Override
                public Type getPayloadType(StompHeaders headers) {
                    return ChatMessage.class;
                }

                @Override
                public void handleFrame(StompHeaders headers, Object payload) {
                    receivedMessages.add((ChatMessage) payload);
                }
            });

        // ส่งข้อความ
        ChatMessage message = new ChatMessage();
        message.setContent("สวัสดี!");
        message.setType(ChatMessage.MessageType.CHAT);

        session.send("/app/chat/room-test", message);

        // รอรับ message
        ChatMessage received = receivedMessages.poll(5, TimeUnit.SECONDS);

        assertThat(received).isNotNull();
        assertThat(received.getContent()).isEqualTo("สวัสดี!");
        assertThat(received.getType()).isEqualTo(ChatMessage.MessageType.CHAT);

        session.disconnect();
    }
}
```

---

## ขั้นตอนที่ 2812: Chat Service และ Message History

```java
// service/ChatService.java
package com.example.realtime.service;

import com.example.realtime.entity.ChatRoom;
import com.example.realtime.entity.Message;
import com.example.realtime.model.ChatMessage;
import com.example.realtime.repository.ChatRoomRepository;
import com.example.realtime.repository.MessageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class ChatService {

    private final MessageRepository messageRepository;
    private final ChatRoomRepository chatRoomRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String ROOM_USERS_PREFIX = "room:users:";
    private static final String RECENT_MESSAGES_PREFIX = "room:messages:";
    private static final int MAX_RECENT_MESSAGES = 50;

    // บันทึก message ลง DB และ cache
    public Message saveMessage(ChatMessage chatMessage) {
        Message entity = Message.builder()
            .roomId(chatMessage.getRoomId())
            .senderId(chatMessage.getSenderId())
            .senderName(chatMessage.getSenderName())
            .content(chatMessage.getContent())
            .type(chatMessage.getType().name())
            .sentAt(LocalDateTime.now())
            .build();

        Message saved = messageRepository.save(entity);

        // Cache ใน Redis สำหรับ quick access
        String cacheKey = RECENT_MESSAGES_PREFIX + chatMessage.getRoomId();
        redisTemplate.opsForList().rightPush(cacheKey, chatMessage);
        redisTemplate.opsForList().trim(cacheKey, -MAX_RECENT_MESSAGES, -1);
        redisTemplate.expire(cacheKey, 24, TimeUnit.HOURS);

        return saved;
    }

    // เพิ่ม user เข้าห้อง
    public void addUserToRoom(String roomId, String userId) {
        redisTemplate.opsForSet().add(ROOM_USERS_PREFIX + roomId, userId);
        redisTemplate.expire(ROOM_USERS_PREFIX + roomId, 24, TimeUnit.HOURS);
    }

    // ดึง users ในห้อง
    @SuppressWarnings("unchecked")
    public Set<String> getUsersInRoom(String roomId) {
        Set<?> members = redisTemplate.opsForSet().members(ROOM_USERS_PREFIX + roomId);
        if (members == null) return Set.of();
        return (Set<String>) (Set<?>) members;
    }

    // ดึง message history
    public List<Message> getMessageHistory(String roomId, int page, int size) {
        return messageRepository.findByRoomIdOrderBySentAtDesc(
            roomId,
            PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "sentAt"))
        ).getContent();
    }

    // ดึง recent messages จาก cache
    @SuppressWarnings("unchecked")
    public List<ChatMessage> getRecentMessages(String roomId) {
        List<?> cached = redisTemplate.opsForList().range(
            RECENT_MESSAGES_PREFIX + roomId, 0, -1
        );
        if (cached == null) return List.of();
        return (List<ChatMessage>) cached;
    }
}
```

---

## สรุป

ในส่วนนี้เราได้เรียนรู้:

1. **WebSocket + STOMP** - สร้าง real-time bidirectional communication
2. **Real-time Chat** - Chat rooms, private messages, typing indicators
3. **Server-Sent Events** - One-way push สำหรับ notifications และ live feeds
4. **Live Dashboard** - Streaming metrics ด้วย Micrometer
5. **Redis Pub/Sub** - Broadcast messages ข้าม server instances
6. **WebSocket Security** - JWT authentication สำหรับ WebSocket connections
7. **Horizontal Scaling** - ใช้ RabbitMQ STOMP relay สำหรับ multi-instance deployment
8. **Presence System** - Track online/offline status ด้วย Redis

---

*[← Part 79: Payment Integration](./part-79-payment-integration.md) | [Part 81: Advanced Patterns →](./part-81-advanced-patterns.md)*

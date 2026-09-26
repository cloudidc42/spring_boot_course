# Part 31: WebSocket & Real-time Features
## ขั้นตอนที่ 856-890

> **ระดับ:** สูง (Advanced)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Real-time communication ด้วย WebSocket และ STOMP

---

## ขั้นตอนที่ 856: WebSocket คืออะไร?

```
HTTP (Traditional):
  Client → Request → Server
  Client ← Response ← Server
  (ต้องส่ง request ใหม่ทุกครั้ง)

WebSocket:
  Client ←→ Server (bidirectional, persistent connection)
  ✅ Real-time chat
  ✅ Live notifications
  ✅ Live dashboard
  ✅ Collaborative editing
  ✅ Online gaming
  ✅ Stock ticker

STOMP (Simple Text Oriented Message Protocol):
  - Protocol เหนือ WebSocket
  - ใช้ publish/subscribe pattern
  - Spring รองรับ built-in
```

---

## ขั้นตอนที่ 857: Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

---

## ขั้นตอนที่ 858: WebSocket Configuration

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // Message broker prefix (destinations for subscribing)
        config.enableSimpleBroker("/topic", "/queue");
        
        // Prefix for messages from client to server
        config.setApplicationDestinationPrefixes("/app");
        
        // Prefix for user-specific messages
        config.setUserDestinationPrefix("/user");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS();  // Fallback for browsers without WebSocket
    }
}
```

---

## ขั้นตอนที่ 859: Chat Application

```java
// Message DTOs
public record ChatMessage(
    String from,
    String content,
    MessageType type,
    String roomId,
    LocalDateTime sentAt
) {
    public enum MessageType { CHAT, JOIN, LEAVE }
}

public record NotificationMessage(
    String title,
    String body,
    String type,
    Object data
) {}

// Chat Controller
@Controller
@RequiredArgsConstructor
@Slf4j
public class ChatController {
    
    private final SimpMessagingTemplate messagingTemplate;
    private final ChatRoomService chatRoomService;
    
    // Client sends to /app/chat.send
    @MessageMapping("/chat.send")
    @SendTo("/topic/chat/{roomId}")
    public ChatMessage sendMessage(
        @Payload ChatMessage message,
        @DestinationVariable String roomId,
        Principal principal
    ) {
        log.info("Message from {} in room {}: {}", principal.getName(), roomId, message.content());
        return new ChatMessage(principal.getName(), message.content(), 
            ChatMessage.MessageType.CHAT, roomId, LocalDateTime.now());
    }
    
    // Handle user joining
    @MessageMapping("/chat.join")
    public void joinRoom(
        @Payload ChatMessage message,
        SimpMessageHeaderAccessor headerAccessor
    ) {
        String username = headerAccessor.getUser().getName();
        String roomId = message.roomId();
        
        headerAccessor.getSessionAttributes().put("username", username);
        headerAccessor.getSessionAttributes().put("roomId", roomId);
        
        chatRoomService.addUser(roomId, username);
        
        ChatMessage joinMessage = new ChatMessage(username, username + " joined the room",
            ChatMessage.MessageType.JOIN, roomId, LocalDateTime.now());
        
        messagingTemplate.convertAndSend("/topic/chat/" + roomId, joinMessage);
    }
    
    // Send private message to specific user
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(
        @Payload ChatMessage message,
        Principal principal
    ) {
        // Sends to /user/{recipient}/queue/private
        messagingTemplate.convertAndSendToUser(
            message.from(),
            "/queue/private",
            new ChatMessage(principal.getName(), message.content(),
                ChatMessage.MessageType.CHAT, null, LocalDateTime.now())
        );
    }
}
```

---

## ขั้นตอนที่ 860: WebSocket Security

```java
@Configuration
public class WebSocketSecurityConfig extends AbstractSecurityWebSocketMessageBrokerConfigurer {
    
    @Override
    protected void configureInbound(MessageSecurityMetadataSourceRegistry messages) {
        messages
            .nullDestMatcher().authenticated()
            .simpSubscribeDestMatchers("/user/**", "/topic/**").authenticated()
            .simpMessageDestMatchers("/app/**").authenticated()
            .anyMessage().denyAll();
    }
    
    @Override
    protected boolean sameOriginDisabled() {
        return true;  // Disable CSRF for WebSocket
    }
}

// JWT auth for WebSocket
@Component
@RequiredArgsConstructor
public class WebSocketAuthInterceptor implements HandshakeInterceptor {
    
    private final JwtService jwtService;
    
    @Override
    public boolean beforeHandshake(
        ServerHttpRequest request, 
        ServerHttpResponse response,
        WebSocketHandler wsHandler,
        Map<String, Object> attributes
    ) {
        if (request instanceof ServletServerHttpRequest servletRequest) {
            String token = servletRequest.getServletRequest().getParameter("token");
            
            if (token != null && jwtService.isTokenValid(token)) {
                String username = jwtService.extractUsername(token);
                attributes.put("username", username);
                return true;
            }
        }
        return false;
    }
    
    @Override
    public void afterHandshake(...) {}
}
```

---

## ขั้นตอนที่ 861: Live Notifications

```java
@Service
@RequiredArgsConstructor
public class NotificationService {
    
    private final SimpMessagingTemplate messagingTemplate;
    
    // Send to specific user
    public void sendToUser(String username, NotificationMessage notification) {
        messagingTemplate.convertAndSendToUser(
            username,
            "/queue/notifications",
            notification
        );
    }
    
    // Send to all users in a topic
    public void broadcast(String topic, NotificationMessage notification) {
        messagingTemplate.convertAndSend("/topic/" + topic, notification);
    }
    
    // After order created - notify user
    public void notifyOrderCreated(Order order) {
        sendToUser(
            order.getUser().getEmail(),
            new NotificationMessage(
                "Order Confirmed",
                "Your order #" + order.getOrderNumber() + " has been confirmed",
                "ORDER_CREATED",
                Map.of("orderId", order.getId())
            )
        );
    }
    
    // Admin dashboard - broadcast to admins
    public void notifyNewOrderToAdmins(Order order) {
        broadcast("admin.orders", new NotificationMessage(
            "New Order",
            "New order #" + order.getOrderNumber() + " from " + order.getUser().getEmail(),
            "NEW_ORDER",
            Map.of("orderId", order.getId(), "amount", order.getTotalAmount())
        ));
    }
}
```

---

## ขั้นตอนที่ 862: Event Listener + WebSocket

```java
@Component
@RequiredArgsConstructor
public class OrderEventListener {
    
    private final NotificationService notificationService;
    
    @EventListener
    @Async
    public void onOrderCreated(OrderCreatedEvent event) {
        notificationService.notifyOrderCreated(event.getOrder());
        notificationService.notifyNewOrderToAdmins(event.getOrder());
    }
}
```

---

## ขั้นตอนที่ 863-890: JavaScript Client

```javascript
// Frontend (React/Vue) WebSocket client
import SockJS from 'sockjs-client';
import { Client } from '@stomp/stompjs';

class WebSocketClient {
    constructor(token) {
        this.token = token;
        this.client = null;
        this.subscriptions = new Map();
    }
    
    connect(onConnected, onDisconnected) {
        this.client = new Client({
            webSocketFactory: () => new SockJS(`/ws?token=${this.token}`),
            
            onConnect: (frame) => {
                console.log('Connected:', frame);
                onConnected?.();
            },
            
            onDisconnect: () => {
                console.log('Disconnected');
                onDisconnected?.();
            },
            
            onStompError: (frame) => {
                console.error('STOMP error:', frame);
            },
            
            reconnectDelay: 5000,
        });
        
        this.client.activate();
    }
    
    disconnect() {
        this.client?.deactivate();
    }
    
    // Subscribe to a topic
    subscribe(destination, callback) {
        const sub = this.client.subscribe(destination, (message) => {
            callback(JSON.parse(message.body));
        });
        this.subscriptions.set(destination, sub);
        return sub;
    }
    
    // Send message
    send(destination, body) {
        this.client.publish({
            destination,
            body: JSON.stringify(body),
        });
    }
    
    // Chat room
    joinRoom(roomId, username) {
        this.send('/app/chat.join', { from: username, roomId, type: 'JOIN' });
        return this.subscribe(`/topic/chat/${roomId}`, (msg) => {
            console.log('New message:', msg);
        });
    }
    
    // Personal notifications
    subscribeToNotifications(callback) {
        return this.subscribe('/user/queue/notifications', callback);
    }
}

// Usage
const ws = new WebSocketClient(localStorage.getItem('token'));
ws.connect(
    () => {
        ws.subscribeToNotifications((notification) => {
            showNotification(notification.title, notification.body);
        });
    }
);
```

---

*[← Part 30: AOP Programming](./part-30-aop-programming.md) | [Part 32: Spring WebFlux →](./part-32-webflux.md)*

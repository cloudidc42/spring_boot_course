# Part 24: Docker & Deployment
## ขั้นตอนที่ 636-665

> **ระดับ:** กลาง-สูง (Intermediate-Advanced)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Containerize และ Deploy Spring Boot application

---

## ขั้นตอนที่ 636: Multi-Stage Dockerfile

```dockerfile
# Multi-stage build สำหรับ production
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app

# Copy Maven wrapper and pom.xml
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies (cache layer)
RUN ./mvnw dependency:go-offline -B

# Copy source and build
COPY src src
RUN ./mvnw package -DskipTests -B

# Extract layers for efficient Docker caching
RUN java -Djarmode=layertools -jar target/*.jar extract

# ===== Final image =====
FROM eclipse-temurin:21-jre-alpine

# Security: create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy layers from builder
COPY --from=builder --chown=appuser:appgroup /app/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /app/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup /app/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /app/application/ ./

USER appuser

# JVM tuning for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+UseG1GC \
               -XX:+UseStringDeduplication \
               -Djava.security.egd=file:/dev/./urandom \
               -Dspring.profiles.active=prod"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

---

## ขั้นตอนที่ 637: Buildpacks (Alternative)

```bash
# Spring Boot 3.x ใช้ Buildpacks ได้เลย
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest

# หรือใน pom.xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>myapp:${project.version}</name>
        </image>
    </configuration>
</plugin>
```

---

## ขั้นตอนที่ 638: docker-compose สำหรับ Production

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:latest
    container_name: spring_app
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DATABASE_URL=jdbc:postgresql://postgres:5432/appdb
      - DATABASE_USERNAME=${DB_USER}
      - DATABASE_PASSWORD=${DB_PASS}
      - REDIS_HOST=redis
      - JWT_SECRET=${JWT_SECRET}
      - MAIL_USERNAME=${MAIL_USERNAME}
      - MAIL_PASSWORD=${MAIL_PASSWORD}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    volumes:
      - uploads:/app/uploads
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 128mb
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf
      - ./config/ssl:/etc/nginx/ssl
      - uploads:/var/www/uploads
    depends_on:
      - app
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
    internal: true  # App network hidden from outside

volumes:
  postgres_data:
  redis_data:
  uploads:
```

---

## ขั้นตอนที่ 639: Nginx Configuration

```nginx
# config/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream spring_app {
        server app:8080;
        keepalive 32;
    }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    server {
        listen 80;
        server_name api.example.com;
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name api.example.com;
        
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
        
        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        
        # Compression
        gzip on;
        gzip_types application/json application/javascript text/css;
        
        # API endpoints
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            
            proxy_pass http://spring_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_connect_timeout 10s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }
        
        # Uploaded files
        location /uploads/ {
            alias /var/www/uploads/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
        
        # Health check (no rate limit)
        location /actuator/health {
            proxy_pass http://spring_app;
        }
        
        # Block actuator in production
        location /actuator/ {
            allow 10.0.0.0/8;  # Internal network only
            deny all;
            proxy_pass http://spring_app;
        }
    }
}
```

---

## ขั้นตอนที่ 640: Environment Configuration

```bash
# .env file (never commit this!)
# Copy from .env.example and fill values

# Database
DB_USER=appuser
DB_PASS=StrongPassword123!

# Redis
REDIS_PASSWORD=RedisPass456!

# Security
JWT_SECRET=your-very-long-256-bit-secret-key-here-at-least-32-chars

# Mail
MAIL_USERNAME=noreply@example.com
MAIL_PASSWORD=MailAppPassword

# AWS (if using S3)
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_S3_BUCKET=my-app-bucket
AWS_S3_REGION=ap-southeast-1

# App
APP_BASE_URL=https://api.example.com
```

```bash
# .env.example (commit this)
DB_USER=change_me
DB_PASS=change_me
REDIS_PASSWORD=change_me
JWT_SECRET=change_me_to_256bit_key
MAIL_USERNAME=change_me
MAIL_PASSWORD=change_me
```

---

## ขั้นตอนที่ 641: Production application.yml

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  
  jpa:
    hibernate.ddl-auto: validate
    show-sql: false
    open-in-view: false
  
  flyway:
    enabled: true
  
  data:
    redis:
      host: ${REDIS_HOST}
      password: ${REDIS_PASSWORD}

server:
  port: 8080
  compression:
    enabled: true
    mime-types: application/json,application/javascript,text/html,text/css
    min-response-size: 1024
  http2:
    enabled: true
  shutdown: graceful  # Graceful shutdown
  servlet:
    context-path: /

logging:
  level:
    root: WARN
    com.yourapp: INFO
  pattern:
    console: "%d{HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: never
```

---

## ขั้นตอนที่ 642: Graceful Shutdown

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # รอสูงสุด 30 วินาที
```

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class GracefulShutdownHook {
    
    @EventListener(ContextClosedEvent.class)
    public void onShutdown() {
        log.info("Application shutting down gracefully...");
        // Cleanup: close connections, finish tasks
    }
    
    @PreDestroy
    public void cleanup() {
        log.info("Cleaning up resources...");
    }
}
```

---

## ขั้นตอนที่ 643-665: CI/CD with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      
      - name: Run tests
        run: ./mvnw test -B
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
  
  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            myapp/spring-boot-app:latest
            myapp/spring-boot-app:${{ github.sha }}
  
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/myapp
            docker-compose pull
            docker-compose up -d --no-deps app
            docker image prune -f
```

---

*[← Part 23: Actuator & Monitoring](./part-23-actuator-monitoring.md) | [Part 25: Microservices Intro →](./part-25-microservices-intro.md)*

# Part 02: ติดตั้งและตั้งค่า Development Environment
## ขั้นตอนที่ 16-35

> **ระดับ:** พื้นฐาน (Beginner)  
> **เวลาเรียน:** 2-3 ชั่วโมง  
> **เป้าหมาย:** ติดตั้งเครื่องมือทั้งหมดและพร้อมเขียนโค้ด Spring Boot

---

## ขั้นตอนที่ 16: เครื่องมือที่ต้องติดตั้ง

```
เครื่องมือที่จำเป็น:
┌────────────────────────────────────────────────────────────┐
│  1. JDK 21 (Java Development Kit)          - บังคับ       │
│  2. IntelliJ IDEA Community Edition        - แนะนำ        │
│  3. Maven 3.9+ หรือ Gradle 8+             - บังคับ        │
│  4. Git                                    - บังคับ        │
│  5. Docker Desktop                         - บังคับ (Part 34+) │
│  6. Postman หรือ Insomnia                  - แนะนำ        │
│  7. DBeaver หรือ TablePlus                 - แนะนำ        │
└────────────────────────────────────────────────────────────┘

Operating System ที่รองรับ:
  ✅ Windows 10/11 (64-bit)
  ✅ macOS 12+ (Intel หรือ Apple Silicon)
  ✅ Linux (Ubuntu 20.04+, Fedora 36+)
```

---

## ขั้นตอนที่ 17: ติดตั้ง JDK 21

### วิธีที่ 1: ดาวน์โหลดโดยตรง (แนะนำสำหรับผู้เริ่มต้น)

**สำหรับ Windows:**
```
1. ไปที่ https://adoptium.net/
2. เลือก Temurin 21 (LTS)
3. ดาวน์โหลด .msi installer
4. ติดตั้งและเลือก "Set JAVA_HOME" และ "Add to PATH"
5. Restart terminal
```

**สำหรับ macOS:**
```bash
# ใช้ Homebrew (แนะนำมาก)
brew install --cask temurin@21

# หรือดาวน์โหลดจาก https://adoptium.net/
```

**สำหรับ Linux (Ubuntu/Debian):**
```bash
# Ubuntu 22.04+
sudo apt update
sudo apt install openjdk-21-jdk

# หรือใช้ SDKMAN (แนะนำมาก)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java 21.0.3-tem
```

### วิธีที่ 2: ใช้ SDKMAN (แนะนำสำหรับนักพัฒนา)

SDKMAN ช่วยให้จัดการ JDK หลายเวอร์ชันได้ง่าย

```bash
# ติดตั้ง SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ดู Java versions ที่มี
sdk list java

# ติดตั้ง Java 21 (Temurin)
sdk install java 21.0.3-tem

# ดู version ที่ติดตั้งแล้ว
sdk list java | grep installed

# เปลี่ยน default version
sdk default java 21.0.3-tem

# ใช้ version ต่างกันในแต่ละ project
sdk use java 17.0.9-tem

# ดู current version
sdk current java
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Java version
java -version
# output: openjdk version "21.0.3" 2024-04-16

# ตรวจสอบ javac
javac -version
# output: javac 21.0.3

# ตรวจสอบ JAVA_HOME
echo $JAVA_HOME  # Linux/macOS
echo %JAVA_HOME%  # Windows

# เนื้อหา JAVA_HOME ควรเป็น path ไปยัง JDK
# เช่น: /usr/lib/jvm/java-21-openjdk-amd64
```

### ตั้งค่า JAVA_HOME (ถ้ายังไม่ได้ตั้ง)

**Linux/macOS - เพิ่มใน ~/.bashrc หรือ ~/.zshrc:**
```bash
# สำหรับ Bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# สำหรับ Zsh (macOS default)
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 21)' >> ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.zshrc
source ~/.zshrc
```

**Windows - System Environment Variables:**
```
1. Win + X → System → Advanced system settings
2. Environment Variables
3. System Variables → New
   Variable: JAVA_HOME
   Value: C:\Program Files\Eclipse Adoptium\jdk-21.0.3.9-hotspot
4. System Variables → Path → Edit → New
   %JAVA_HOME%\bin
5. Restart Command Prompt
```

---

## ขั้นตอนที่ 18: ติดตั้ง IntelliJ IDEA

### ดาวน์โหลดและติดตั้ง

```
1. ไปที่ https://www.jetbrains.com/idea/
2. เลือก Community Edition (ฟรี) หรือ Ultimate (มีฟีเจอร์เพิ่มเติม)
   - Community: ครอบคลุมสำหรับผู้เริ่มต้น
   - Ultimate: มี Spring-specific features, Database tools, HTTP Client
3. ดาวน์โหลดและติดตั้ง
```

### Plugins ที่แนะนำ

```
File → Settings → Plugins → Marketplace

ติดตั้ง Plugins เหล่านี้:
┌─────────────────────────────────────────────────────────┐
│  1. Spring Boot Assistant     - Spring hints & completion │
│  2. Lombok                    - ลด boilerplate code      │
│  3. Docker                    - Docker integration       │
│  4. .env files support        - สำหรับ .env files        │
│  5. Rainbow Brackets          - อ่านโค้ดง่ายขึ้น         │
│  6. GitToolBox                - Git enhancements         │
│  7. Indent Rainbow            - Indent visualization     │
│  8. HTTP Client               - Built-in REST client     │
│  9. Database Tools            - (Ultimate only)          │
│  10. Checkstyle-IDEA          - Code style checker       │
└─────────────────────────────────────────────────────────┘
```

### ตั้งค่า IntelliJ IDEA

```
File → Settings (Ctrl+Alt+S)

1. Build, Execution, Deployment → Build Tools → Maven
   - Maven home path: (ชี้ไปที่ Maven ที่ติดตั้ง)
   - User settings file: ~/.m2/settings.xml

2. Build, Execution, Deployment → Compiler
   - Build project automatically: ✓ เปิด
   - Annotation Processing: ✓ เปิด (สำหรับ Lombok, MapStruct)

3. Editor → General → Auto Import
   - Add unambiguous imports on the fly: ✓
   - Optimize imports on the fly: ✓

4. Editor → Code Style → Java
   - Scheme: Default IDE Scheme หรือ Google Style

5. Editor → Inspections
   - Spring: ✓ เปิด Spring inspections ทั้งหมด

6. Tools → Terminal
   - Shell path: /bin/zsh หรือ /bin/bash
```

### IntelliJ Shortcuts ที่ใช้บ่อย

```
Shift+Shift              → Search everything
Ctrl+N / Cmd+O          → Open class
Ctrl+Shift+N / Cmd+Shift+O → Open file
Ctrl+Alt+L / Cmd+Alt+L  → Format code
Ctrl+Alt+O / Cmd+Alt+O  → Optimize imports
Alt+Enter               → Quick fix
Ctrl+Space              → Code completion
Ctrl+P                  → Parameter info
Ctrl+Q                  → Quick documentation
Ctrl+B / Cmd+B          → Go to definition
Alt+F7                  → Find usages
Ctrl+R / Cmd+R          → Replace
Ctrl+F9 / Cmd+F9        → Build project
Shift+F10               → Run application
Shift+F9                → Debug application
Ctrl+Shift+F10          → Run current file
```

---

## ขั้นตอนที่ 19: ติดตั้ง Maven

Maven เป็น build tool ที่ใช้จัดการ dependencies และ build project

### ดาวน์โหลดและติดตั้ง

**วิธีที่ 1: ดาวน์โหลดโดยตรง**
```bash
# Linux/macOS
wget https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
tar xzf apache-maven-3.9.6-bin.tar.gz
sudo mv apache-maven-3.9.6 /opt/maven

# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
export M2_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}
source ~/.bashrc
```

**วิธีที่ 2: Package Manager**
```bash
# macOS
brew install maven

# Ubuntu/Debian
sudo apt install maven

# Windows (ใช้ Chocolatey)
choco install maven
```

**วิธีที่ 3: SDKMAN**
```bash
sdk install maven 3.9.6
```

### ตรวจสอบการติดตั้ง

```bash
mvn --version
# output: Apache Maven 3.9.6
#         Maven home: /opt/maven
#         Java version: 21.0.3
```

### Maven Configuration (~/.m2/settings.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
          http://maven.apache.org/xsd/settings-1.0.0.xsd">
    
    <!-- Maven local repository -->
    <localRepository>${user.home}/.m2/repository</localRepository>
    
    <!-- Mirror ในประเทศไทย (เร็วกว่า) -->
    <mirrors>
        <mirror>
            <id>central</id>
            <mirrorOf>central</mirrorOf>
            <url>https://repo1.maven.org/maven2</url>
        </mirror>
    </mirrors>
    
    <!-- Proxy settings (ถ้าอยู่หลัง corporate proxy) -->
    <!--
    <proxies>
        <proxy>
            <id>company-proxy</id>
            <active>true</active>
            <protocol>http</protocol>
            <host>proxy.company.com</host>
            <port>8080</port>
        </proxy>
    </proxies>
    -->
</settings>
```

---

## ขั้นตอนที่ 20: ติดตั้ง Git

```bash
# macOS
brew install git

# Ubuntu/Debian
sudo apt install git

# Windows
# ดาวน์โหลดจาก https://git-scm.com/download/win

# ตรวจสอบ
git --version
# output: git version 2.43.0

# ตั้งค่าพื้นฐาน
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global core.editor "vim"  # หรือ "code" สำหรับ VS Code

# สร้าง SSH key สำหรับ GitHub
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
# Copy ไปใส่ GitHub → Settings → SSH Keys
```

---

## ขั้นตอนที่ 21: ติดตั้ง Docker Desktop

Docker ใช้สำหรับรัน Database, Redis, และ Services ต่างๆ โดยไม่ต้องติดตั้งบนเครื่อง

### ติดตั้ง

```
macOS / Windows:
1. ไปที่ https://www.docker.com/products/docker-desktop
2. ดาวน์โหลดและติดตั้ง Docker Desktop
3. เปิด Docker Desktop และรอให้ Engine start

Linux (Ubuntu):
```

```bash
# ติดตั้ง Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# เพิ่ม user ปัจจุบันไปยัง docker group
sudo usermod -aG docker $USER
newgrp docker

# ติดตั้ง Docker Compose
sudo apt install docker-compose-plugin

# ตรวจสอบ
docker --version
docker compose version
```

### ทดสอบ Docker

```bash
# รัน hello-world
docker run hello-world

# รัน PostgreSQL สำหรับ development
docker run --name postgres-dev \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -d postgres:16

# ตรวจสอบ container ที่กำลังรัน
docker ps

# หยุด container
docker stop postgres-dev

# ลบ container
docker rm postgres-dev
```

---

## ขั้นตอนที่ 22: ติดตั้ง Postman

Postman ใช้ทดสอบ REST API

```
1. ไปที่ https://www.postman.com/downloads/
2. ดาวน์โหลดและติดตั้ง
3. สร้าง account (ฟรี)
4. สร้าง Collection สำหรับ project

Alternative - ใช้ curl บน Terminal:
curl -X GET http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN"

Alternative - IntelliJ IDEA HTTP Client:
# สร้างไฟล์ .http ใน project
### Get all users
GET http://localhost:8080/api/users
Content-Type: application/json
Authorization: Bearer TOKEN

### Create user
POST http://localhost:8080/api/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

---

## ขั้นตอนที่ 23: ติดตั้ง Database Client

### DBeaver (ฟรี, รองรับหลาย database)

```
1. ไปที่ https://dbeaver.io/download/
2. เลือก Community Edition (ฟรี)
3. ดาวน์โหลดและติดตั้ง
4. เชื่อมต่อกับ database:
   - Host: localhost
   - Port: 5432 (PostgreSQL) หรือ 3306 (MySQL)
   - User: admin
   - Password: secret
   - Database: myapp
```

---

## ขั้นตอนที่ 24: ใช้ Spring Initializr สร้างโปรเจค

Spring Initializr เป็นเว็บไซต์ที่ช่วยสร้าง Spring Boot project template

### วิธีที่ 1: เว็บไซต์ https://start.spring.io

```
การตั้งค่าแนะนำ:
┌─────────────────────────────────────────────────────────┐
│  Project:     Maven                                      │
│  Language:    Java                                       │
│  Spring Boot: 3.3.x (latest stable)                     │
│                                                          │
│  Project Metadata:                                       │
│  Group:    com.example                                   │
│  Artifact: myapp                                         │
│  Name:     myapp                                         │
│  Package:  com.example.myapp                             │
│  Packaging: Jar                                          │
│  Java:     21                                            │
│                                                          │
│  Dependencies:                                           │
│  ✓ Spring Web                                           │
│  ✓ Spring Data JPA                                      │
│  ✓ PostgreSQL Driver                                    │
│  ✓ Spring Boot DevTools                                 │
│  ✓ Lombok                                               │
│  ✓ Validation                                           │
└─────────────────────────────────────────────────────────┘
```

### วิธีที่ 2: IntelliJ IDEA

```
File → New Project → Spring Boot
(IntelliJ จะใช้ Spring Initializr โดยอัตโนมัติ)
```

### วิธีที่ 3: Spring Boot CLI

```bash
# ติดตั้ง Spring Boot CLI ด้วย SDKMAN
sdk install springboot

# สร้าง project
spring init \
  --boot-version=3.3.4 \
  --build=maven \
  --java-version=21 \
  --dependencies=web,data-jpa,postgresql,devtools,lombok,validation \
  --group-id=com.example \
  --artifact-id=myapp \
  --name=myapp \
  myapp
```

### วิธีที่ 4: Maven Archetype

```bash
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=myapp \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false

cd myapp
# แล้วแก้ pom.xml เพิ่ม Spring Boot ด้วยตนเอง
```

---

## ขั้นตอนที่ 25: โครงสร้าง Maven Project

```
myapp/
├── .mvn/
│   └── wrapper/
│       ├── maven-wrapper.jar
│       └── maven-wrapper.properties
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── myapp/
│   │   │               ├── MyappApplication.java    ← Main class
│   │   │               ├── controller/
│   │   │               ├── service/
│   │   │               ├── repository/
│   │   │               ├── model/
│   │   │               ├── dto/
│   │   │               └── config/
│   │   └── resources/
│   │       ├── application.properties               ← Config file
│   │       ├── application-dev.properties           ← Dev config
│   │       ├── application-prod.properties          ← Prod config
│   │       └── static/                              ← Static files
│   └── test/
│       └── java/
│           └── com/
│               └── example/
│                   └── myapp/
│                       ├── MyappApplicationTests.java
│                       ├── controller/
│                       └── service/
├── .gitignore
├── mvnw                                             ← Maven Wrapper (Linux/Mac)
├── mvnw.cmd                                         ← Maven Wrapper (Windows)
└── pom.xml                                          ← Project config
```

---

## ขั้นตอนที่ 26: ทำความเข้าใจ pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <!-- POM Version (always 4.0.0) -->
    <modelVersion>4.0.0</modelVersion>
    
    <!-- Spring Boot Parent - จัดการ versions ให้ทั้งหมด -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <!-- Project Identification -->
    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>myapp</name>
    <description>My Spring Boot Application</description>
    
    <!-- Java Version -->
    <properties>
        <java.version>21</java.version>
    </properties>
    
    <!-- Dependencies -->
    <dependencies>
        
        <!-- Web (Spring MVC + Embedded Tomcat) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- JPA (Hibernate + Spring Data) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- PostgreSQL Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Lombok (ลด boilerplate code) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Bean Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- DevTools (Hot reload) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
    </dependencies>
    
    <!-- Build Configuration -->
    <build>
        <plugins>
            <!-- Spring Boot Maven Plugin - สร้าง executable JAR -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!-- Exclude Lombok จาก final JAR -->
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
    
</project>
```

---

## ขั้นตอนที่ 27: Maven Commands ที่ใช้บ่อย

```bash
# Build project (compile + test + package)
mvn clean install

# เฉพาะ compile
mvn compile

# รัน tests
mvn test

# Build เป็น JAR โดยข้าม tests
mvn clean package -DskipTests

# รัน application
mvn spring-boot:run

# ดู dependencies tree
mvn dependency:tree

# ดู available updates
mvn versions:display-dependency-updates

# Generate project reports
mvn site

# Download dependencies (เพื่อใช้ offline)
mvn dependency:go-offline

# ล้าง build artifacts
mvn clean

# ใช้ Maven Wrapper (แทนที่จะใช้ mvn โดยตรง)
./mvnw spring-boot:run    # Linux/macOS
mvnw.cmd spring-boot:run  # Windows
```

---

## ขั้นตอนที่ 28: ตั้งค่า application.properties

```properties
# ========================================
# Application Configuration
# ========================================

# Application Name
spring.application.name=myapp

# Server Port (default: 8080)
server.port=8080

# Context Path (default: /)
# server.servlet.context-path=/api

# ========================================
# Database Configuration
# ========================================

# PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/myapp
spring.datasource.username=admin
spring.datasource.password=secret
spring.datasource.driver-class-name=org.postgresql.Driver

# Connection Pool (HikariCP - default)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000

# ========================================
# JPA / Hibernate Configuration
# ========================================

# DDL Auto: none, validate, update, create, create-drop
spring.jpa.hibernate.ddl-auto=update

# Show SQL queries in logs
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Dialect
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect

# ========================================
# Logging Configuration
# ========================================

# Log level: TRACE, DEBUG, INFO, WARN, ERROR
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate.SQL=DEBUG

# Log file
logging.file.name=logs/app.log
logging.file.max-size=10MB
logging.file.max-history=30

# ========================================
# Jackson (JSON) Configuration
# ========================================

# Date format
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=Asia/Bangkok

# null fields จะไม่ถูก serialize
spring.jackson.default-property-inclusion=non_null

# ========================================
# DevTools Configuration
# ========================================

# Hot reload
spring.devtools.restart.enabled=true
spring.devtools.livereload.enabled=true
```

### application.yml (แบบ YAML - อ่านง่ายกว่า)

```yaml
spring:
  application:
    name: myapp
  
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: admin
    password: secret
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
    database-platform: org.hibernate.dialect.PostgreSQLDialect
  
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Bangkok
    default-property-inclusion: non_null

server:
  port: 8080

logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework.web: DEBUG
```

---

## ขั้นตอนที่ 29: ติดตั้ง Docker สำหรับ Database

สร้างไฟล์ `docker-compose.yml` สำหรับ development:

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: myapp-postgres
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./docker/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    command: redis-server --requirepass redispassword
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # pgAdmin (Database GUI)
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: myapp-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres-data:
  redis-data:
```

### รัน Docker Compose

```bash
# รัน ทุก services
docker compose up -d

# รัน เฉพาะ postgres
docker compose up -d postgres

# ดู logs
docker compose logs -f

# หยุด services
docker compose down

# หยุดและลบ volumes (ระวัง! ข้อมูลจะหาย)
docker compose down -v

# รัน SQL command ใน postgres
docker exec -it myapp-postgres psql -U admin -d myapp
```

---

## ขั้นตอนที่ 30: ตั้งค่า .gitignore

```gitignore
# Maven
target/
!.mvn/wrapper/maven-wrapper.jar
!**/src/main/**/target/
!**/src/test/**/target/

# Gradle (ถ้าใช้)
.gradle
build/

# IDE
.idea/
*.iws
*.iml
*.ipr
.vscode/
*.classpath
*.project
.settings/

# OS
.DS_Store
Thumbs.db

# Spring Boot
HELP.md
*.log
logs/

# Environment files (สำคัญมาก! ห้าม commit)
.env
.env.local
.env.*.local
application-local.properties
application-local.yml

# Secrets (สำคัญมาก! ห้าม commit)
*.p12
*.jks
*.key
*.pem
*.crt
secrets/

# Docker
docker-compose.override.yml
```

---

## ขั้นตอนที่ 31: เข้าใจ Spring Boot DevTools

DevTools ทำให้ development เร็วขึ้นด้วย:

```xml
<!-- เพิ่มใน pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

### Features ของ DevTools

```
1. Automatic Restart
   - เมื่อ classpath เปลี่ยน, app restart อัตโนมัติ
   - เร็วกว่า cold restart เพราะใช้ 2 ClassLoaders
   - Base ClassLoader (libraries ที่ไม่เปลี่ยน)
   - Restart ClassLoader (code ที่เราเขียน)

2. LiveReload
   - Browser refresh อัตโนมัติเมื่อ resources เปลี่ยน
   - ต้องติดตั้ง LiveReload extension ใน browser

3. Property Defaults
   - ปิด template caching
   - DEBUG logging สำหรับ web layer
   - H2 console เปิดอัตโนมัติ (ถ้าใช้ H2)

4. Remote Development
   - Remote debugging ผ่าน HTTP tunnel
```

### ปิด/เปิด DevTools features

```properties
# ปิด automatic restart
spring.devtools.restart.enabled=false

# เพิ่ม directories ที่ trigger restart
spring.devtools.restart.additional-paths=scripts

# ยกเว้น paths จาก restart trigger
spring.devtools.restart.exclude=static/**,public/**

# ปิด LiveReload
spring.devtools.livereload.enabled=false
```

---

## ขั้นตอนที่ 32: ใช้ Lombok ลด Boilerplate Code

```xml
<!-- เพิ่มใน pom.xml -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### Lombok Annotations

```java
import lombok.*;

// @Getter - สร้าง getter methods ทุก field
// @Setter - สร้าง setter methods ทุก field
// @ToString - สร้าง toString()
// @EqualsAndHashCode - สร้าง equals() และ hashCode()
// @NoArgsConstructor - สร้าง no-args constructor
// @AllArgsConstructor - สร้าง all-args constructor
// @RequiredArgsConstructor - สร้าง constructor สำหรับ final fields

// @Data = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
@Data
public class User {
    private Long id;
    private String name;
    private String email;
}

// @Builder - สร้าง Builder pattern
@Builder
@Data
public class UserDTO {
    private Long id;
    private String name;
    private String email;
    private String role;
}

// ใช้งาน Builder
UserDTO user = UserDTO.builder()
    .id(1L)
    .name("John Doe")
    .email("john@example.com")
    .role("ADMIN")
    .build();

// @Value - Immutable class (ทุก field เป็น final)
@Value
public class ApiResponse {
    String message;
    Object data;
    int status;
}

// @Slf4j - สร้าง SLF4J logger
@Slf4j
@Service
public class UserService {
    
    public User findById(Long id) {
        log.info("Finding user with id: {}", id);  // ใช้ log โดยตรง!
        // ...
    }
}

// @SneakyThrows - Wrap checked exception
@SneakyThrows
public void processFile(String path) {
    Files.readAllLines(Path.of(path));  // IOException จะถูก wrap อัตโนมัติ
}
```

---

## ขั้นตอนที่ 33: ตั้งค่า Profiles สำหรับ Development

```
development/
├── application.properties          ← Common config
├── application-dev.properties      ← Development config
├── application-test.properties     ← Test config
└── application-prod.properties     ← Production config
```

```properties
# application.properties (ทุก environment ใช้ร่วมกัน)
spring.application.name=myapp

# เลือก profile
spring.profiles.active=dev
```

```properties
# application-dev.properties
spring.datasource.url=jdbc:postgresql://localhost:5432/myapp_dev
spring.datasource.username=admin
spring.datasource.password=secret
spring.jpa.show-sql=true
logging.level.com.example=DEBUG

# DevTools
spring.devtools.restart.enabled=true
```

```properties
# application-prod.properties
spring.datasource.url=${DATABASE_URL}
spring.datasource.username=${DATABASE_USERNAME}
spring.datasource.password=${DATABASE_PASSWORD}
spring.jpa.show-sql=false
logging.level.com.example=WARN

# ปิด DevTools ใน production
spring.devtools.restart.enabled=false
```

### รัน Application ด้วย Profile ต่างๆ

```bash
# Dev profile (default ตาม application.properties)
mvn spring-boot:run

# Explicit profile
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Production profile
java -jar myapp.jar --spring.profiles.active=prod

# หรือผ่าน environment variable
SPRING_PROFILES_ACTIVE=prod java -jar myapp.jar
```

---

## ขั้นตอนที่ 34: ตั้งค่า IDE สำหรับ Hot Reload

```
IntelliJ IDEA settings สำหรับ Hot Reload:

1. Settings → Build, Execution, Deployment → Compiler
   ✓ Build project automatically

2. Settings → Advanced Settings
   ✓ Allow auto-make to start even if developed application is currently running

3. หรือใช้ JRebel Plugin (paid, ทรงพลังกว่า DevTools)
```

### เปิด Annotation Processing (สำหรับ Lombok)

```
Settings → Build, Execution, Deployment → Compiler → Annotation Processors
✓ Enable annotation processing
```

---

## ขั้นตอนที่ 35: สรุป Environment Setup

### Checklist ก่อนเริ่มเรียน Part 03

```
✅ JDK 21 ติดตั้งแล้ว
   $ java -version
   openjdk version "21.x.x"

✅ Maven ติดตั้งแล้ว
   $ mvn --version
   Apache Maven 3.9.x

✅ Git ติดตั้งแล้ว
   $ git --version
   git version 2.x.x

✅ IntelliJ IDEA ติดตั้งแล้ว พร้อม Plugins
   - Lombok plugin
   - Spring Boot Assistant

✅ Docker Desktop ติดตั้งแล้ว
   $ docker --version
   Docker version 25.x.x

✅ Postman ติดตั้งแล้ว (หรือ curl ทำงานได้)

✅ DBeaver ติดตั้งแล้ว (optional)
```

### สรุป Commands ที่ต้องจำ

```bash
# Maven
mvn spring-boot:run          # รัน application
mvn clean install            # Build ทั้งหมด
mvn clean package -DskipTests # Build ข้าม tests
mvn test                     # รัน tests

# Docker
docker compose up -d         # รัน services
docker compose down          # หยุด services
docker compose logs -f       # ดู logs

# Git
git init                     # เริ่ม repository
git add .                    # Stage all files
git commit -m "message"      # Commit
git push origin main         # Push
```

---

> ✅ Environment พร้อมแล้ว! ไปเรียน Part 03 กัน

*[← Part 01: บทนำ](./part-01-introduction.md) | [Part 03: สร้าง Application แรก →](./part-03-first-application.md)*

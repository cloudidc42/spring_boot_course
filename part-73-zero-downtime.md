# Part 73: Zero-Downtime Deployment
## ขั้นตอนที่ 2521-2560

**ระดับ:** ระดับโลก (World-Class)
**เวลาเรียน:** 5-6 ชั่วโมง
**เป้าหมาย:** เรียนรู้เทคนิค Zero-Downtime Deployment ตั้งแต่ Blue-Green, Canary, Database Migration, Feature Flags และ Health Check Gates สำหรับระบบ Production ที่ต้องการ availability สูง

---

## สารบัญ

1. [Zero-Downtime Concepts](#concepts)
2. [Blue-Green Deployment with Kubernetes](#blue-green)
3. [Canary Deployment with Istio](#canary)
4. [Database Migration Strategies](#db-migration)
5. [Feature Flags for Dark Launches](#feature-flags)
6. [Traffic Shifting and Rollback](#traffic-shifting)
7. [Health Check Gates](#health-check-gates)

---

## ขั้นตอนที่ 2521: Zero-Downtime Deployment คืออะไร? {#concepts}

### ปัญหาของ Traditional Deployment

```
Traditional (ทำให้ downtime):
Time 10:00 → Stop old version → Deploy new version → Start → Time 10:05
              ↑ DOWNTIME 5 minutes ↑

Zero-Downtime:
Time 10:00 → Start new version alongside old → Shift traffic → Remove old
              ↑ Always available ↑
```

### ทำไม Spring Boot ต้องระวังเป็นพิเศษ?

1. **JVM Startup Time** - Spring Boot อาจ start ช้า (อาจ 30-60 วินาที)
2. **Database Schema Changes** - Migration ที่ไม่ดีทำให้ old version พัง
3. **In-flight Requests** - Request ที่กำลัง process อยู่เมื่อ pod ถูก terminate
4. **Session State** - Session ที่เก็บใน memory จะหายเมื่อ pod ถูก stop

### Graceful Shutdown Configuration

```yaml
# application.yml
server:
  shutdown: graceful  # ไม่รับ request ใหม่ แต่รอให้ request เดิม finish

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # รอ request เดิม 30 วินาที ก่อน force shutdown
```

```java
package com.example.config;

import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class GracefulShutdownHandler implements ApplicationListener<ContextClosedEvent> {

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("Application shutdown initiated - completing in-flight requests...");
        // ทำ cleanup tasks เช่น:
        // - ปิด Kafka consumer gracefully
        // - รอ async tasks เสร็จ
        // - บันทึก state ถ้าจำเป็น
    }
}
```

---

## ขั้นตอนที่ 2522: Spring Boot Actuator สำหรับ Kubernetes

```java
package com.example.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.boot.actuate.availability.LivenessState;
import org.springframework.boot.actuate.availability.ReadinessState;
import org.springframework.boot.availability.ApplicationAvailability;
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

@Component
public class ApplicationReadinessManager {

    private final ApplicationEventPublisher publisher;

    public ApplicationReadinessManager(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    // เรียกเมื่อ application พร้อมรับ traffic
    public void markReady() {
        AvailabilityChangeEvent.publish(publisher, this, ReadinessState.ACCEPTING_TRAFFIC);
    }

    // เรียกเมื่อต้องการหยุดรับ traffic (เช่น ก่อน shutdown)
    public void markNotReady() {
        AvailabilityChangeEvent.publish(publisher, this, ReadinessState.REFUSING_TRAFFIC);
    }

    // เรียกเมื่อ application มีปัญหาร้ายแรง (ต้องการให้ Kubernetes restart)
    public void markBroken() {
        AvailabilityChangeEvent.publish(publisher, this, LivenessState.BROKEN);
    }
}
```

### Kubernetes Probes Configuration

```yaml
# k8s-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # เพิ่ม pod ได้สูงสุด 1 ขณะ deploy
      maxUnavailable: 0  # ห้าม pod ใดใด unavailable ขณะ deploy
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          ports:
            - containerPort: 8080
          
          # Liveness Probe - ถ้า fail → restart container
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30  # รอ 30s หลัง start
            periodSeconds: 10
            failureThreshold: 3     # fail 3 ครั้ง ค่อย restart
            
          # Readiness Probe - ถ้า fail → หยุดส่ง traffic
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
            
          # Startup Probe - ให้เวลา startup มากขึ้น
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30  # รวมสูงสุด 30*10 = 300s
            periodSeconds: 10
            
          # Resources limits
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          
          # Lifecycle hooks
          lifecycle:
            preStop:
              exec:
                # รอ load balancer ถอด pod ออกก่อน
                command: ["/bin/sh", "-c", "sleep 10"]
      
      # Graceful termination
      terminationGracePeriodSeconds: 60
```

---

## ขั้นตอนที่ 2525: Blue-Green Deployment with Kubernetes {#blue-green}

### แนวคิด Blue-Green

```
[Load Balancer]
       ↓
  [Blue Service] ← Current (100% traffic)
  [Green Service] ← New version (0% traffic, testing)

After verification:
[Load Balancer]
       ↓
  [Blue Service] ← Old (0% traffic, standby)
  [Green Service] ← New version (100% traffic)
```

### Kubernetes Blue-Green Manifests

```yaml
# blue-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app-blue
  labels:
    app: spring-boot-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-app
      version: blue
  template:
    metadata:
      labels:
        app: spring-boot-app
        version: blue
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          ports:
            - containerPort: 8080
```

```yaml
# green-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app-green
  labels:
    app: spring-boot-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-app
      version: green
  template:
    metadata:
      labels:
        app: spring-boot-app
        version: green
    spec:
      containers:
        - name: app
          image: myapp:2.0.0  # New version
          ports:
            - containerPort: 8080
```

```yaml
# service.yml - ชี้ไปที่ blue หรือ green
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-app
spec:
  selector:
    app: spring-boot-app
    version: blue  # เปลี่ยนเป็น 'green' เพื่อ switch traffic
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Blue-Green Deployment Script

```bash
#!/bin/bash
# deploy-blue-green.sh

set -e

APP_NAME="spring-boot-app"
NAMESPACE="production"
NEW_VERSION=$1
NEW_IMAGE="myapp:${NEW_VERSION}"

# ตรวจสอบว่า version ไหนกำลัง active อยู่
CURRENT_COLOR=$(kubectl get service $APP_NAME -n $NAMESPACE \
  -o jsonpath='{.spec.selector.version}')

if [ "$CURRENT_COLOR" == "blue" ]; then
    NEW_COLOR="green"
    OLD_COLOR="blue"
else
    NEW_COLOR="blue"
    OLD_COLOR="green"
fi

echo "Current: $OLD_COLOR → Deploying to: $NEW_COLOR"

# Deploy version ใหม่ไปยัง inactive color
echo "1. Deploying $NEW_IMAGE to $NEW_COLOR..."
kubectl set image deployment/${APP_NAME}-${NEW_COLOR} \
    app=$NEW_IMAGE -n $NAMESPACE

# รอให้ deployment เสร็จ
echo "2. Waiting for $NEW_COLOR to be ready..."
kubectl rollout status deployment/${APP_NAME}-${NEW_COLOR} \
    -n $NAMESPACE --timeout=5m

# Run smoke tests
echo "3. Running smoke tests against $NEW_COLOR..."
NEW_COLOR_IP=$(kubectl get pod -n $NAMESPACE \
    -l "app=$APP_NAME,version=$NEW_COLOR" \
    -o jsonpath='{.items[0].status.podIP}')

HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    "http://$NEW_COLOR_IP:8080/actuator/health")

if [ "$HEALTH_STATUS" != "200" ]; then
    echo "ERROR: Smoke test failed! Health check returned $HEALTH_STATUS"
    exit 1
fi

# Switch traffic
echo "4. Switching traffic from $OLD_COLOR to $NEW_COLOR..."
kubectl patch service $APP_NAME -n $NAMESPACE \
    -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_COLOR\"}}}"

# Verify traffic switch
echo "5. Verifying traffic switch..."
sleep 5  # รอ a bit

# Optional: Scale down old version
echo "6. Scaling down $OLD_COLOR (keeping for rollback)..."
kubectl scale deployment/${APP_NAME}-${OLD_COLOR} \
    --replicas=0 -n $NAMESPACE

echo "Deployment completed! Traffic is now routing to $NEW_COLOR ($NEW_VERSION)"
echo "To rollback: kubectl patch service $APP_NAME -n $NAMESPACE -p '{\"spec\":{\"selector\":{\"version\":\"$OLD_COLOR\"}}}'"
```

---

## ขั้นตอนที่ 2530: Canary Deployment with Istio {#canary}

### Istio VirtualService สำหรับ Traffic Splitting

```yaml
# istio-canary.yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: spring-boot-app
spec:
  hosts:
    - spring-boot-app
  http:
    - route:
        - destination:
            host: spring-boot-app
            subset: stable  # Current version
          weight: 90        # 90% traffic
        - destination:
            host: spring-boot-app
            subset: canary  # New version
          weight: 10        # 10% traffic (canary)

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: spring-boot-app
spec:
  host: spring-boot-app
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

### Progressive Traffic Shifting Script

```bash
#!/bin/bash
# canary-progressive.sh

NAMESPACE="production"
SERVICE="spring-boot-app"

# ค่อยๆ เพิ่ม traffic ไป canary
for CANARY_WEIGHT in 10 25 50 75 100; do
    STABLE_WEIGHT=$((100 - CANARY_WEIGHT))
    
    echo "Shifting traffic: stable=$STABLE_WEIGHT%, canary=$CANARY_WEIGHT%"
    
    # Update VirtualService weights
    kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: $SERVICE
  namespace: $NAMESPACE
spec:
  hosts:
    - $SERVICE
  http:
    - route:
        - destination:
            host: $SERVICE
            subset: stable
          weight: $STABLE_WEIGHT
        - destination:
            host: $SERVICE
            subset: canary
          weight: $CANARY_WEIGHT
EOF
    
    # รอ และตรวจสอบ metrics
    echo "Waiting 2 minutes to observe metrics..."
    sleep 120
    
    # ตรวจสอบ error rate ของ canary
    ERROR_RATE=$(kubectl exec -n istio-system deploy/prometheus -- \
        curl -s "http://localhost:9090/api/v1/query?query=\
        sum(rate(istio_requests_total{destination_service_name=\"$SERVICE\",\
        destination_version=\"canary\",response_code=~\"5..\"}[2m]))\
        /\
        sum(rate(istio_requests_total{destination_service_name=\"$SERVICE\",\
        destination_version=\"canary\"}[2m]))" | \
        jq -r '.data.result[0].value[1] // "0"')
    
    echo "Canary error rate: $ERROR_RATE"
    
    # ถ้า error rate > 5% ให้ rollback
    if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
        echo "ERROR: Canary error rate too high! Rolling back..."
        kubectl apply -f - <<ROLLBACK
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: $SERVICE
  namespace: $NAMESPACE
spec:
  hosts:
    - $SERVICE
  http:
    - route:
        - destination:
            host: $SERVICE
            subset: stable
          weight: 100
        - destination:
            host: $SERVICE
            subset: canary
          weight: 0
ROLLBACK
        exit 1
    fi
done

echo "Canary deployment successful! 100% traffic now on canary."
```

### Automated Canary Analysis (Flagger)

```yaml
# flagger-canary.yml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: spring-boot-app
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-boot-app
  
  progressDeadlineSeconds: 600
  
  service:
    port: 8080
    targetPort: 8080
    gateways:
      - public-gateway.istio-system.svc.cluster.local
    hosts:
      - app.example.com
  
  analysis:
    interval: 1m
    threshold: 5        # ล้มเหลวได้ 5 ครั้งก่อน rollback
    maxWeight: 50       # สูงสุด 50% traffic ให้ canary
    stepWeight: 10      # เพิ่มทีละ 10%
    
    # Metrics ที่ใช้ตัดสินใจ
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99       # ต้องมี success rate >= 99%
        interval: 1m
      
      - name: request-duration
        thresholdRange:
          max: 500      # latency ต้องไม่เกิน 500ms
        interval: 1m
    
    # Webhook สำหรับ smoke test ก่อน promote
    webhooks:
      - name: smoke-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://spring-boot-app-canary/api/health"
      
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          type: bash
          cmd: "hey -z 1m -q 10 -c 2 http://spring-boot-app-canary/api/orders"
```

---

## ขั้นตอนที่ 2535: Database Migration Strategies {#db-migration}

### Expand-Contract Pattern

```
WRONG (breaking):
v1: table has column "amount"
v2: rename to "total_amount" → v1 code breaks!

CORRECT (Expand-Contract):
Phase 1 (Expand): ADD column "total_amount", keep "amount"
Phase 2 (Migrate): Copy data, update app to use "total_amount"
Phase 3 (Contract): DROP column "amount" (after all old pods gone)
```

### Flyway Migration Scripts

```sql
-- V1__Create_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
```

```sql
-- V2__Add_total_amount_column.sql (Expand Phase - Non-breaking)
-- เพิ่ม column ใหม่ ไม่ลบ column เก่า
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);

-- Copy data จาก column เก่า
UPDATE orders SET total_amount = amount WHERE total_amount IS NULL;

-- ทำ NOT NULL หลังจาก copy data เสร็จ
ALTER TABLE orders ALTER COLUMN total_amount SET NOT NULL;

-- สร้าง trigger เพื่อ sync ทั้งสอง column (ระหว่างที่ app กำลัง migrate)
CREATE OR REPLACE FUNCTION sync_order_amounts()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.amount IS DISTINCT FROM OLD.amount THEN
        NEW.total_amount = NEW.amount;
    ELSIF NEW.total_amount IS DISTINCT FROM OLD.total_amount THEN
        NEW.amount = NEW.total_amount;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_amounts_trigger
    BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION sync_order_amounts();
```

```sql
-- V3__Drop_amount_column.sql (Contract Phase - Run AFTER all old pods gone)
-- ลบ trigger ก่อน
DROP TRIGGER IF EXISTS sync_amounts_trigger ON orders;
DROP FUNCTION IF EXISTS sync_order_amounts();

-- ลบ column เก่า
ALTER TABLE orders DROP COLUMN amount;
```

### Flyway Configuration

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    # ใช้ repair เมื่อ checksum ไม่ตรง
    repair-on-migrate: false
    # Out-of-order migration สำหรับ hotfix
    out-of-order: false
    # Placeholder สำหรับ environment-specific SQL
    placeholders:
      schema: orders
      tablespace: pg_default
```

### Zero-Downtime Migration ด้วย Liquibase

```xml
<!-- src/main/resources/db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- Expand: Add new column -->
    <changeSet id="2024-01-expand-add-total-amount" author="developer">
        <addColumn tableName="orders">
            <column name="total_amount" type="DECIMAL(10,2)">
                <constraints nullable="true"/>
            </column>
        </addColumn>
        
        <!-- Copy existing data -->
        <update tableName="orders">
            <column name="total_amount" valueComputed="amount"/>
            <where>total_amount IS NULL</where>
        </update>
        
        <!-- Make not null after backfill -->
        <addNotNullConstraint 
            tableName="orders" 
            columnName="total_amount"
            defaultNullValue="0"/>
        
        <rollback>
            <dropColumn tableName="orders" columnName="total_amount"/>
        </rollback>
    </changeSet>

    <!-- Contract: Remove old column (separate deployment) -->
    <changeSet id="2024-02-contract-remove-amount" author="developer">
        <preConditions onFail="MARK_RAN">
            <!-- ทำแค่เมื่อ column เก่ายังมีอยู่ -->
            <columnExists tableName="orders" columnName="amount"/>
        </preConditions>
        <dropColumn tableName="orders" columnName="amount"/>
    </changeSet>

</databaseChangeLog>
```

---

## ขั้นตอนที่ 2540: Feature Flags for Dark Launches {#feature-flags}

### Togglz Feature Flags

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-spring-boot-starter</artifactId>
    <version>3.3.3</version>
</dependency>
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-spring-security</artifactId>
    <version>3.3.3</version>
</dependency>
```

```java
package com.example.feature;

import org.togglz.core.Feature;
import org.togglz.core.annotation.EnabledByDefault;
import org.togglz.core.annotation.Label;
import org.togglz.core.context.FeatureContext;

// กำหนด features ทั้งหมดในที่เดียว
public enum AppFeatures implements Feature {

    @Label("New Order Recommendation Engine")
    NEW_RECOMMENDATION_ENGINE,

    @Label("Real-time Inventory Check")
    REALTIME_INVENTORY,

    @Label("Dark Launch: New Payment Flow")
    NEW_PAYMENT_FLOW,

    @EnabledByDefault
    @Label("Legacy Order Processing")
    LEGACY_ORDER_PROCESSING,

    @Label("Beta: AI Price Optimization")
    AI_PRICE_OPTIMIZATION;

    public boolean isActive() {
        return FeatureContext.getFeatureManager().isActive(this);
    }
}
```

### Feature Flag Configuration

```yaml
# application.yml
togglz:
  features:
    NEW_RECOMMENDATION_ENGINE:
      enabled: false
    REALTIME_INVENTORY:
      enabled: true
    NEW_PAYMENT_FLOW:
      enabled: false
    LEGACY_ORDER_PROCESSING:
      enabled: true
    AI_PRICE_OPTIMIZATION:
      enabled: false
  # ใช้ database เพื่อ persist state
  feature-manager-stub: false
```

```java
package com.example.service;

import com.example.feature.AppFeatures;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    public List<Product> getRecommendations(String customerId) {
        // Dark launch - test new engine ด้วย subset ของ users
        if (AppFeatures.NEW_RECOMMENDATION_ENGINE.isActive()) {
            try {
                return newRecommendationEngine.getRecommendations(customerId);
            } catch (Exception e) {
                log.error("New recommendation engine failed, falling back", e);
                return legacyRecommendationEngine.getRecommendations(customerId);
            }
        }
        return legacyRecommendationEngine.getRecommendations(customerId);
    }

    public PaymentResult processPayment(PaymentRequest request) {
        if (AppFeatures.NEW_PAYMENT_FLOW.isActive()) {
            return newPaymentFlow.process(request);
        }
        return legacyPaymentFlow.process(request);
    }
}
```

### Percentage-Based Rollout

```java
package com.example.feature;

import org.togglz.core.manager.FeatureManager;
import org.togglz.core.repository.FeatureState;
import org.togglz.core.user.FeatureUser;
import org.togglz.core.activation.ActivationStrategy;
import org.togglz.core.spi.ActivationStrategyProvider;
import org.springframework.stereotype.Component;

@Component
public class GradualRolloutStrategy implements ActivationStrategy {

    @Override
    public String getId() {
        return "gradual_rollout";
    }

    @Override
    public String getName() {
        return "Gradual Rollout by User ID";
    }

    @Override
    public boolean isActive(FeatureState featureState, FeatureUser user) {
        String percentageStr = featureState.getParameter("percentage");
        if (percentageStr == null) return false;
        
        int percentage = Integer.parseInt(percentageStr);
        
        // กำหนด hash ของ user ID เพื่อให้ consistent
        // (user เดิมจะได้รับ feature เดิมทุกครั้ง)
        int userHash = Math.abs(user.getName().hashCode() % 100);
        return userHash < percentage;
    }

    @Override
    public Parameter[] getParameters() {
        return new Parameter[]{
            ActivationStrategyUtils.newIntRangeParameter(
                "percentage", 
                "Percentage of users to enable feature for",
                0, 100)
        };
    }
}
```

### LaunchDarkly Integration (Enterprise)

```java
package com.example.feature;

import com.launchdarkly.sdk.LDContext;
import com.launchdarkly.sdk.server.LDClient;
import org.springframework.stereotype.Service;

@Service
public class FeatureFlagService {

    private final LDClient ldClient;

    public FeatureFlagService(@Value("${launchdarkly.sdk.key}") String sdkKey) {
        this.ldClient = new LDClient(sdkKey);
    }

    public boolean isFeatureEnabled(String featureKey, String userId) {
        LDContext context = LDContext.builder(userId)
            .set("email", getUserEmail(userId))
            .set("country", getUserCountry(userId))
            .build();
        
        return ldClient.boolVariation(featureKey, context, false);
    }

    public String getFeatureVariant(String featureKey, String userId, String defaultVariant) {
        LDContext context = LDContext.builder(userId).build();
        return ldClient.stringVariation(featureKey, context, defaultVariant);
    }

    // A/B Test
    public int getExperimentVariant(String experimentKey, String userId) {
        LDContext context = LDContext.builder(userId).build();
        return ldClient.intVariation(experimentKey, context, 0);
    }
}
```

---

## ขั้นตอนที่ 2545: Traffic Shifting and Rollback {#traffic-shifting}

### Automated Rollback Controller

```java
package com.example.deployment;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class DeploymentHealthMonitor {

    private final MeterRegistry meterRegistry;
    private final KubernetesClient kubernetesClient;
    private final AlertService alertService;

    private static final double ERROR_RATE_THRESHOLD = 0.05; // 5%
    private static final double LATENCY_THRESHOLD_MS = 1000.0; // 1s P99

    @Scheduled(fixedDelay = 30000) // ทุก 30 วินาที
    public void monitorDeploymentHealth() {
        // ตรวจสอบ canary deployment ถ้ามี
        DeploymentInfo canary = kubernetesClient.getCanaryDeployment();
        if (canary == null) return;

        double errorRate = getErrorRate(canary.getVersion());
        double p99Latency = getP99Latency(canary.getVersion());

        log.info("Canary health: errorRate={}%, p99={}ms", 
            errorRate * 100, p99Latency);

        if (errorRate > ERROR_RATE_THRESHOLD) {
            log.error("Canary error rate {} exceeds threshold {}! Initiating rollback...",
                errorRate, ERROR_RATE_THRESHOLD);
            initiateRollback(canary, "High error rate: " + errorRate);
            return;
        }

        if (p99Latency > LATENCY_THRESHOLD_MS) {
            log.error("Canary P99 latency {}ms exceeds threshold {}ms! Rolling back...",
                p99Latency, LATENCY_THRESHOLD_MS);
            initiateRollback(canary, "High latency: " + p99Latency + "ms");
        }
    }

    private void initiateRollback(DeploymentInfo deployment, String reason) {
        try {
            // Rollback Kubernetes deployment
            kubernetesClient.rollback(deployment.getName());
            
            // ส่ง alert
            alertService.sendAlert(Alert.builder()
                .severity("CRITICAL")
                .message("Automatic rollback triggered for " + deployment.getName())
                .reason(reason)
                .build());
            
            log.info("Rollback completed for {}", deployment.getName());
        } catch (Exception e) {
            log.error("Rollback failed for {}!", deployment.getName(), e);
            alertService.sendCriticalAlert("ROLLBACK FAILED for " + deployment.getName());
        }
    }

    private double getErrorRate(String version) {
        return meterRegistry.find("http.server.requests")
            .tags("version", version, "status", "5xx")
            .timer()
            .map(timer -> {
                double errorCount = timer.count();
                double totalCount = meterRegistry.find("http.server.requests")
                    .tags("version", version)
                    .timer()
                    .map(t -> (double) t.count())
                    .orElse(1.0);
                return errorCount / totalCount;
            })
            .orElse(0.0);
    }

    private double getP99Latency(String version) {
        return meterRegistry.find("http.server.requests")
            .tags("version", version)
            .timer()
            .map(timer -> timer.percentile(0.99) * 1000)  // convert to ms
            .orElse(0.0);
    }
}
```

---

## ขั้นตอนที่ 2550: Health Check Gates in Deployment Pipeline {#health-check-gates}

### Pre-Deployment Gate

```java
package com.example.deployment;

import org.springframework.boot.actuate.health.CompositeHealth;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthEndpoint;
import org.springframework.stereotype.Component;

@Component
public class DeploymentGateChecker {

    private final HealthEndpoint healthEndpoint;
    private final MetricsValidator metricsValidator;

    // ตรวจสอบว่า system พร้อม deploy หรือยัง
    public GateResult checkPreDeploymentGate() {
        List<String> failures = new ArrayList<>();

        // Gate 1: ทุก dependencies healthy
        Health systemHealth = healthEndpoint.health();
        if (!systemHealth.getStatus().equals(Status.UP)) {
            failures.add("System health is not UP: " + systemHealth.getStatus());
        }

        // Gate 2: Error rate ต่ำพอ
        double currentErrorRate = metricsValidator.getCurrentErrorRate();
        if (currentErrorRate > 0.01) {  // ต้อง < 1%
            failures.add("Current error rate too high: " + currentErrorRate);
        }

        // Gate 3: ไม่มี ongoing incidents
        if (incidentTracker.hasActiveIncidents()) {
            failures.add("There are active incidents - deployment blocked");
        }

        // Gate 4: ช่วงเวลา deploy (ห้าม deploy ช่วง peak hours)
        LocalTime now = LocalTime.now();
        LocalTime peakStart = LocalTime.of(18, 0);
        LocalTime peakEnd = LocalTime.of(22, 0);
        if (now.isAfter(peakStart) && now.isBefore(peakEnd)) {
            failures.add("Cannot deploy during peak hours (18:00-22:00)");
        }

        return failures.isEmpty() 
            ? GateResult.passed() 
            : GateResult.failed(failures);
    }

    // ตรวจสอบหลัง deploy
    public GateResult checkPostDeploymentGate(String newVersion) {
        List<String> failures = new ArrayList<>();

        // รอให้ pods ready
        try {
            Thread.sleep(30000); // 30 seconds
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // Gate 1: Readiness probe pass
        // Gate 2: Error rate ไม่เพิ่มขึ้น
        double newErrorRate = metricsValidator.getErrorRate(newVersion, Duration.ofMinutes(2));
        if (newErrorRate > 0.05) {
            failures.add("New version error rate too high: " + newErrorRate);
        }

        // Gate 3: Latency ไม่แย่ลง
        double newP99 = metricsValidator.getP99Latency(newVersion, Duration.ofMinutes(2));
        if (newP99 > 1000) { // 1 second
            failures.add("New version P99 latency too high: " + newP99 + "ms");
        }

        return failures.isEmpty()
            ? GateResult.passed()
            : GateResult.failed(failures);
    }
}
```

### CI/CD Pipeline with Gates

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true

jobs:
  pre-deployment-gate:
    name: Pre-Deployment Gate Check
    runs-on: ubuntu-latest
    steps:
      - name: Check system health
        run: |
          HEALTH=$(curl -s https://api.production.com/actuator/health | jq -r '.status')
          if [ "$HEALTH" != "UP" ]; then
            echo "System health check failed: $HEALTH"
            exit 1
          fi

      - name: Check error rate
        run: |
          ERROR_RATE=$(curl -s "https://prometheus.internal/api/v1/query?\
            query=sum(rate(http_requests_total{status=~\"5..\"}[5m]))/sum(rate(http_requests_total[5m]))" | \
            jq -r '.data.result[0].value[1]')
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            exit 1
          fi

      - name: Check deployment window
        run: |
          HOUR=$(date -u +%H)
          if [ $HOUR -ge 18 ] && [ $HOUR -le 22 ]; then
            echo "Cannot deploy during peak hours"
            exit 1
          fi

  deploy:
    name: Deploy Version ${{ github.event.inputs.version }}
    needs: pre-deployment-gate
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/spring-boot-app \
            app=myapp:${{ github.event.inputs.version }} \
            -n production
          
          kubectl rollout status deployment/spring-boot-app \
            -n production --timeout=5m

  post-deployment-gate:
    name: Post-Deployment Verification
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: Wait for stabilization
        run: sleep 60

      - name: Verify error rate
        run: |
          ERROR_RATE=$(curl -s "https://prometheus.internal/api/v1/query?\
            query=sum(rate(http_requests_total{status=~\"5..\",version=\"${{ github.event.inputs.version }}\"}[2m]))/\
            sum(rate(http_requests_total{version=\"${{ github.event.inputs.version }}\"}[2m]))" | \
            jq -r '.data.result[0].value[1] // "0"')
          
          if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
            echo "Post-deployment error rate too high: $ERROR_RATE - initiating rollback!"
            kubectl rollout undo deployment/spring-boot-app -n production
            exit 1
          fi
          
          echo "Deployment successful! Error rate: $ERROR_RATE"

      - name: Update deployment tracking
        run: |
          curl -X POST https://deployment-tracker.internal/api/deployments \
            -H "Content-Type: application/json" \
            -d '{
              "version": "${{ github.event.inputs.version }}",
              "status": "SUCCESS",
              "deployedAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
            }'
```

---

## ขั้นตอนที่ 2555: Database Migration สำหรับ Zero-Downtime

### Spring Boot Migration Configuration

```java
package com.example.config;

import org.flywaydb.core.Flyway;
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationStrategy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import lombok.extern.slf4j.Slf4j;

@Configuration
@Slf4j
public class DatabaseMigrationConfig {

    // Custom migration strategy
    @Bean
    public FlywayMigrationStrategy flywayMigrationStrategy() {
        return flyway -> {
            log.info("Starting database migration...");
            
            // Validate migrations ก่อน
            try {
                flyway.validate();
                log.info("Migration validation passed");
            } catch (Exception e) {
                log.warn("Migration validation failed, attempting repair: {}", e.getMessage());
                flyway.repair();
            }
            
            // Run migrations
            int applied = flyway.migrate().migrationsExecuted;
            log.info("Applied {} migrations", applied);
        };
    }
}
```

```java
package com.example.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.PreparedStatement;

// Java-based migration สำหรับ complex logic
public class V4__Backfill_order_categories extends BaseJavaMigration {

    @Override
    public void migrate(Context context) throws Exception {
        // Process in batches เพื่อไม่ lock table นาน
        int batchSize = 1000;
        int processed = 0;
        
        try (PreparedStatement countStmt = context.getConnection()
                .prepareStatement("SELECT COUNT(*) FROM orders WHERE category IS NULL")) {
            
            var rs = countStmt.executeQuery();
            rs.next();
            int total = rs.getInt(1);
            
            System.out.println("Backfilling " + total + " orders...");
        }
        
        // Process batch by batch
        String updateSql = """
            UPDATE orders
            SET category = CASE
                WHEN total_amount < 100 THEN 'SMALL'
                WHEN total_amount < 1000 THEN 'MEDIUM'
                ELSE 'LARGE'
            END
            WHERE id IN (
                SELECT id FROM orders 
                WHERE category IS NULL 
                ORDER BY id 
                LIMIT ?
            )
            """;
        
        int updated;
        do {
            try (PreparedStatement stmt = context.getConnection().prepareStatement(updateSql)) {
                stmt.setInt(1, batchSize);
                updated = stmt.executeUpdate();
                processed += updated;
                System.out.println("Processed " + processed + " orders...");
                
                // Small sleep เพื่อลด database pressure
                if (updated > 0) Thread.sleep(100);
            }
        } while (updated > 0);
        
        System.out.println("Backfill complete: " + processed + " orders updated");
    }
}
```

---

## สรุปสิ่งที่เรียนรู้

ใน Part นี้เราได้เรียนรู้:

1. **Graceful Shutdown** - การตั้งค่า Spring Boot ให้ shutdown อย่างถูกต้อง
2. **Blue-Green Deployment** - การ switch traffic ระหว่าง 2 environments
3. **Canary Deployment** - การค่อยๆ เพิ่ม traffic ให้ version ใหม่
4. **Database Migration** - Expand-Contract pattern สำหรับ zero-downtime schema changes
5. **Feature Flags** - Dark launch และ gradual rollout
6. **Traffic Shifting** - Automated rollback เมื่อ metrics แย่ลง
7. **Health Check Gates** - Pre/Post deployment validation

### Best Practices
- ตั้งค่า `server.shutdown: graceful` เสมอ
- ใช้ `maxUnavailable: 0` ใน rolling update strategy
- Database migration: Expand ก่อน, Contract ทีหลัง
- Feature flags สำหรับ risky changes ก่อน deploy
- Automated rollback เมื่อ error rate เกิน threshold

---

*[← Part 72: Contract Testing](./part-72-contract-testing.md) | [Part 74: Event-Driven Advanced →](./part-74-event-driven-advanced.md)*

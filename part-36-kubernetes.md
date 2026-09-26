# Part 36: Kubernetes Deployment
## ขั้นตอนที่ 1041-1085

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 8-10 ชั่วโมง  
> **เป้าหมาย:** Deploy Spring Boot ใน Kubernetes production

---

## ขั้นตอนที่ 1041: Kubernetes Concepts

```
Kubernetes (K8s) = Container orchestration platform

Key Objects:
  Pod         = Container(s) ที่รันด้วยกัน (smallest deployable unit)
  Deployment  = Manages Pod replicas, rolling updates
  Service     = Network endpoint for Pods (load balancing)
  ConfigMap   = Non-sensitive config
  Secret      = Sensitive config (encrypted)
  Ingress     = HTTP routing, TLS termination
  PVC         = Persistent Volume Claim (storage)
  HPA         = Horizontal Pod Autoscaler

Architecture:
  Control Plane:
    API Server    = Gateway to cluster
    etcd          = State store
    Scheduler     = Assigns pods to nodes
    Controller Manager = Maintains desired state
  
  Worker Nodes:
    kubelet     = Node agent
    kube-proxy  = Network proxy
    Container runtime = Docker/containerd
```

---

## ขั้นตอนที่ 1042: Dockerfile Optimization for K8s

```dockerfile
# Multi-stage build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -q

COPY src src
RUN ./mvnw package -DskipTests -q

# Extract layers for better caching
RUN java -Djarmode=layertools -jar target/*.jar extract

# Runtime image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Non-root user for security
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## ขั้นตอนที่ 1043: Kubernetes Manifests

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app: myapp

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-service:5432/mydb"
  SPRING_REDIS_HOST: "redis-service"
  SPRING_REDIS_PORT: "6379"
  SERVER_PORT: "8080"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
data:
  # Base64 encoded values
  SPRING_DATASOURCE_USERNAME: cG9zdGdyZXM=
  SPRING_DATASOURCE_PASSWORD: c2VjcmV0UGFzc3dvcmQ=
  JWT_SECRET: bXlTdXBlclNlY3JldEtleUZvckpXVA==

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:1.0.0
          ports:
            - containerPort: 8080
          
          # Resource limits
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          
          # Environment from ConfigMap
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      
      terminationGracePeriodSeconds: 60
```

---

## ขั้นตอนที่ 1044: Service and Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

---

## ขั้นตอนที่ 1045: Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## ขั้นตอนที่ 1046: Spring Boot Actuator for K8s

```yaml
# application-prod.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, redis
  
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health, info, prometheus, metrics

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  shutdown: graceful
```

```java
// Custom readiness health indicator
@Component
public class KafkaHealthIndicator extends AbstractHealthIndicator {
    
    private final KafkaAdmin kafkaAdmin;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            Map<String, Object> details = kafkaAdmin.describeCluster();
            builder.up().withDetails(details);
        } catch (Exception ex) {
            builder.down(ex);
        }
    }
}
```

---

## ขั้นตอนที่ 1047: Database in Kubernetes

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_DB
              value: mydb
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
        storageClassName: fast-ssd

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: myapp
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
  type: ClusterIP
  clusterIP: None  # Headless for StatefulSet
```

---

## ขั้นตอนที่ 1048-1085: kubectl Commands

```bash
# Deploy
kubectl apply -f k8s/

# Check status
kubectl get pods -n myapp
kubectl get deployments -n myapp
kubectl get services -n myapp

# Logs
kubectl logs -f deployment/myapp -n myapp
kubectl logs pod/myapp-xxx-yyy -n myapp --previous  # crashed pod

# Exec into pod
kubectl exec -it pod/myapp-xxx-yyy -n myapp -- /bin/sh

# Port forward for local testing
kubectl port-forward svc/myapp-service 8080:80 -n myapp

# Scale manually
kubectl scale deployment myapp --replicas=5 -n myapp

# Rolling update
kubectl set image deployment/myapp myapp=myregistry/myapp:2.0.0 -n myapp
kubectl rollout status deployment/myapp -n myapp

# Rollback
kubectl rollout undo deployment/myapp -n myapp
kubectl rollout history deployment/myapp -n myapp

# Check HPA
kubectl get hpa -n myapp
kubectl describe hpa myapp-hpa -n myapp

# Resource usage
kubectl top pods -n myapp
kubectl top nodes

# Describe for debugging
kubectl describe pod/myapp-xxx-yyy -n myapp
kubectl describe deployment/myapp -n myapp

# Delete
kubectl delete pod/myapp-xxx-yyy -n myapp  # Pod restarts automatically
kubectl delete deployment/myapp -n myapp   # Removes all pods
```

---

*[← Part 35: OAuth2 & OIDC](./part-35-oauth2.md) | [Part 37: OpenTelemetry →](./part-37-opentelemetry.md)*

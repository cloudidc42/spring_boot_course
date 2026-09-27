# Part 92: DevOps Best Practices for Spring Boot
## ขั้นตอนที่ 3281-3320

**ระดับ:** World-Class (ระดับโลก)
**เวลาเรียน:** 6-8 ชั่วโมง
**เป้าหมาย:** เรียนรู้ DevOps practices ระดับ production สำหรับ Spring Boot applications รวมถึง GitOps ด้วย ArgoCD, Helm Charts, Multi-environment pipelines, Secret management ด้วย Vault, Infrastructure as Code ด้วย Terraform และ Automated rollback

---

## ขั้นตอนที่ 3281: GitOps พื้นฐานและ ArgoCD

GitOps คือแนวทาง DevOps ที่ใช้ Git เป็น single source of truth สำหรับ infrastructure และ application deployment ArgoCD ทำหน้าที่ sync state ระหว่าง Git repository กับ Kubernetes cluster อัตโนมัติ

### ทำไมต้องใช้ GitOps?

- **Auditability** - ทุกการเปลี่ยนแปลงถูกบันทึกใน Git
- **Rollback ง่าย** - แค่ revert commit ก็ rollback ได้
- **Consistency** - environment ทุกตัวมาจาก source เดียวกัน
- **Security** - ไม่ต้องให้ CI/CD เข้าถึง cluster โดยตรง

### ติดตั้ง ArgoCD

```bash
# สร้าง namespace สำหรับ ArgoCD
kubectl create namespace argocd

# ติดตั้ง ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอจนกว่า pods จะพร้อม
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s

# Port forward เพื่อเข้าถึง UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ดูรหัสผ่านเริ่มต้น
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### สร้าง ArgoCD Application

```yaml
# argocd/applications/shophub-user-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shophub-user-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: shophub
  source:
    repoURL: https://github.com/myorg/shophub-gitops
    targetRevision: HEAD
    path: kubernetes/user-service
  destination:
    server: https://kubernetes.default.svc
    namespace: shophub-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
```

```yaml
# argocd/projects/shophub-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: shophub
  namespace: argocd
spec:
  description: ShopHub Microservices Platform
  sourceRepos:
    - 'https://github.com/myorg/shophub-gitops'
    - 'https://charts.helm.sh/stable'
  destinations:
    - namespace: shophub-dev
      server: https://kubernetes.default.svc
    - namespace: shophub-staging
      server: https://kubernetes.default.svc
    - namespace: shophub-prod
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  roles:
    - name: developer
      description: Developer access
      policies:
        - p, proj:shophub:developer, applications, get, shophub/*, allow
        - p, proj:shophub:developer, applications, sync, shophub/dev-*, allow
      groups:
        - developers
```

## ขั้นตอนที่ 3282: Helm Chart สำหรับ Spring Boot

Helm ช่วยให้จัดการ Kubernetes manifests ได้อย่างเป็นระบบ สามารถ version และ share charts ได้

### โครงสร้าง Helm Chart

```
shophub-chart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    ├── pdb.yaml
    └── serviceaccount.yaml
```

```yaml
# shophub-chart/Chart.yaml
apiVersion: v2
name: shophub-service
description: A generic Helm chart for ShopHub microservices
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

```yaml
# shophub-chart/values.yaml
# Default values for shophub-service
replicaCount: 1

image:
  repository: myregistry.azurecr.io/shophub
  pullPolicy: IfNotPresent
  tag: ""

imagePullSecrets:
  - name: acr-secret

serviceAccount:
  create: true
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: api.shophub.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

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

podDisruptionBudget:
  enabled: true
  minAvailable: 1

env:
  SPRING_PROFILES_ACTIVE: kubernetes

envFrom:
  - configMapRef:
      name: shophub-config
  - secretRef:
      name: shophub-secrets

postgresql:
  enabled: false

nodeSelector: {}
tolerations: []
affinity: {}
```

```yaml
# shophub-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "shophub-service.fullname" . }}
  labels:
    {{- include "shophub-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "shophub-service.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
      labels:
        {{- include "shophub-service.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "shophub-service.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            {{- toYaml .Values.envFrom | nindent 12 }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 60
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

```yaml
# shophub-chart/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "shophub-service.fullname" . }}
  labels:
    {{- include "shophub-service.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "shophub-service.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
{{- end }}
```

## ขั้นตอนที่ 3283: Multi-Environment Promotion Pipeline

Pipeline ที่ promote artifact จาก dev → staging → production อย่างปลอดภัย

```yaml
# .github/workflows/ci-cd-pipeline.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: shophub/user-service

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Run tests
        run: mvn test -pl user-service --no-transfer-progress

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: Run SAST scan
        uses: anchore/scan-action@v3
        with:
          path: "."
          fail-build: true
          severity-cutoff: high

  build-and-push:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build JAR
        run: mvn package -pl user-service -DskipTests --no-transfer-progress

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: ./user-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}

      - name: Sign image with cosign
        uses: sigstore/cosign-installer@v3
        with:
          cosign-release: 'v2.2.0'
      - run: |
          cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

  deploy-dev:
    name: Deploy to Dev
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://dev.shophub.internal
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/shophub-gitops
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update image tag in dev
        run: |
          IMAGE_TAG=$(echo "${{ needs.build-and-push.outputs.image-tag }}" | head -1)
          cd kubernetes/user-service/overlays/dev
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}=$IMAGE_TAG
          git config user.email "ci@shophub.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "chore: update user-service to $IMAGE_TAG in dev"
          git push

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    environment:
      name: staging
      url: https://staging.shophub.internal
    steps:
      - name: Wait for dev deployment health
        run: |
          # รอให้ ArgoCD sync เสร็จ
          sleep 60
          # ตรวจสอบ health ด้วย smoke test
          curl -f https://dev.shophub.internal/actuator/health || exit 1

      - uses: actions/checkout@v4
        with:
          repository: myorg/shophub-gitops
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Promote image to staging
        run: |
          IMAGE_TAG=$(echo "${{ needs.build-and-push.outputs.image-tag }}" | head -1)
          cd kubernetes/user-service/overlays/staging
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}=$IMAGE_TAG
          git config user.email "ci@shophub.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "chore: promote user-service to $IMAGE_TAG in staging"
          git push

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Run integration tests against staging
        run: |
          mvn verify -pl integration-tests \
            -Dtest.base.url=https://staging.shophub.internal \
            --no-transfer-progress

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.shophub.com
    steps:
      - uses: actions/checkout@v4
        with:
          repository: myorg/shophub-gitops
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Promote image to production
        run: |
          IMAGE_TAG=$(echo "${{ needs.build-and-push.outputs.image-tag }}" | head -1)
          cd kubernetes/user-service/overlays/prod
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}=$IMAGE_TAG
          git config user.email "ci@shophub.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "chore: promote user-service to $IMAGE_TAG in prod"
          git push
```

## ขั้นตอนที่ 3284: Secret Management ด้วย HashiCorp Vault

Vault ช่วยจัดเก็บและจัดการ secrets อย่างปลอดภัย แทนที่การใช้ Kubernetes secrets ธรรมดา

### ติดตั้ง Vault บน Kubernetes

```bash
# เพิ่ม Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# ติดตั้ง Vault
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.ha.enabled=true" \
  --set "server.ha.replicas=3"

# Initialize Vault
kubectl exec -it vault-0 -n vault -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-keys.json

# Unseal (ต้องใช้ key อย่างน้อย 3 จาก 5)
VAULT_KEYS=$(cat vault-keys.json | jq -r '.unseal_keys_b64[]')
for key in $(echo "$VAULT_KEYS" | head -3); do
  kubectl exec -it vault-0 -n vault -- vault operator unseal $key
done
```

### ตั้งค่า Vault สำหรับ Kubernetes Authentication

```bash
# เปิดใช้ Kubernetes auth method
vault auth enable kubernetes

# ตั้งค่า Kubernetes auth
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT_HTTPS" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# สร้าง policy สำหรับ user-service
vault policy write user-service - <<EOF
path "secret/data/shophub/user-service/*" {
  capabilities = ["read", "list"]
}
path "database/creds/user-service" {
  capabilities = ["read"]
}
EOF

# สร้าง role สำหรับ user-service
vault write auth/kubernetes/role/user-service \
  bound_service_account_names=user-service \
  bound_service_account_namespaces=shophub-prod \
  policies=user-service \
  ttl=1h
```

### เก็บ Secrets ใน Vault

```bash
# เก็บ application secrets
vault kv put secret/shophub/user-service \
  jwt-secret="super-secret-jwt-key-2024" \
  smtp-password="smtp-pass-123" \
  encryption-key="aes-256-key-here"

# ตั้งค่า Dynamic Database Credentials
vault secrets enable database

vault write database/config/user-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="user-service" \
  connection_url="postgresql://{{username}}:{{password}}@postgres-user:5432/user_db" \
  username="vault-admin" \
  password="vault-admin-pass"

vault write database/roles/user-service \
  db_name=user-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

### Spring Boot Integration กับ Vault

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml
spring:
  cloud:
    vault:
      uri: http://vault:8200
      authentication: KUBERNETES
      kubernetes:
        role: user-service
        kubernetes-path: kubernetes
      kv:
        enabled: true
        backend: secret
        default-context: shophub/user-service
      database:
        enabled: true
        role: user-service
        backend: database
```

```java
// ใช้ secrets ใน application
@Configuration
public class DatabaseConfig {

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    // Vault จะ inject dynamic credentials อัตโนมัติ
    // Spring Cloud Vault จะ refresh credentials ก่อนหมดอายุ
}
```

### Vault Agent Sidecar

```yaml
# kubernetes/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "user-service"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/shophub/user-service"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/shophub/user-service" -}}
          SPRING_SECURITY_JWT_SECRET={{ .Data.data.jwt-secret }}
          SPRING_MAIL_PASSWORD={{ .Data.data.smtp-password }}
          {{- end }}
    spec:
      serviceAccountName: user-service
      containers:
        - name: user-service
          env:
            - name: SPRING_CONFIG_IMPORT
              value: "configtree:/vault/secrets/"
```

## ขั้นตอนที่ 3285: Infrastructure as Code ด้วย Terraform

Terraform ช่วยสร้างและจัดการ infrastructure บน cloud อย่างเป็นระบบ

### โครงสร้าง Terraform

```
terraform/
├── modules/
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   ├── elasticache/
│   └── msk/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
└── shared/
    └── backend.tf
```

```hcl
# terraform/modules/eks/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = var.public_access
    public_access_cidrs     = var.public_access_cidrs
  }

  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  tags = var.tags
}

# Node Group สำหรับ application workloads
resource "aws_eks_node_group" "application" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "application"
  node_role_arn   = aws_iam_role.nodes.arn
  subnet_ids      = var.private_subnet_ids

  instance_types = var.node_instance_types
  capacity_type  = "ON_DEMAND"

  scaling_config {
    desired_size = var.node_desired_size
    max_size     = var.node_max_size
    min_size     = var.node_min_size
  }

  update_config {
    max_unavailable = 1
  }

  launch_template {
    id      = aws_launch_template.nodes.id
    version = aws_launch_template.nodes.latest_version
  }

  labels = {
    role = "application"
  }

  taint {
    key    = "role"
    value  = "application"
    effect = "NO_SCHEDULE"
  }

  tags = var.tags
}

# Node Group สำหรับ Spot instances (batch workloads)
resource "aws_eks_node_group" "spot" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "spot"
  node_role_arn   = aws_iam_role.nodes.arn
  subnet_ids      = var.private_subnet_ids

  instance_types = ["m5.large", "m5a.large", "m4.large"]
  capacity_type  = "SPOT"

  scaling_config {
    desired_size = 0
    max_size     = 20
    min_size     = 0
  }

  labels = {
    role = "spot"
  }

  taint {
    key    = "spot"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  tags = var.tags
}
```

```hcl
# terraform/modules/rds/main.tf
# RDS Aurora PostgreSQL สำหรับ production
resource "aws_rds_cluster" "main" {
  cluster_identifier      = var.cluster_id
  engine                  = "aurora-postgresql"
  engine_version          = "15.4"
  database_name           = var.database_name
  master_username         = var.master_username
  manage_master_user_password = true  # Secrets Manager จัดการ password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"

  deletion_protection     = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "${var.cluster_id}-final-snapshot"

  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  enabled_cloudwatch_logs_exports = ["postgresql"]

  serverlessv2_scaling_configuration {
    max_capacity = var.max_capacity
    min_capacity = var.min_capacity
  }

  tags = var.tags
}

resource "aws_rds_cluster_instance" "main" {
  count              = var.instance_count
  identifier         = "${var.cluster_id}-${count.index}"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version

  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
}
```

```hcl
# terraform/environments/prod/main.tf
terraform {
  required_version = ">= 1.6"

  backend "s3" {
    bucket         = "shophub-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = "production"
      Project     = "shophub"
      ManagedBy   = "terraform"
    }
  }
}

module "eks" {
  source = "../../modules/eks"

  cluster_name       = "shophub-prod"
  kubernetes_version = "1.29"
  subnet_ids         = module.vpc.subnet_ids
  private_subnet_ids = module.vpc.private_subnet_ids

  node_instance_types  = ["m5.xlarge"]
  node_desired_size    = 3
  node_min_size        = 2
  node_max_size        = 20

  tags = local.tags
}

module "rds_user" {
  source = "../../modules/rds"

  cluster_id    = "shophub-user-prod"
  database_name = "user_db"
  instance_count = 2
  min_capacity   = 0.5
  max_capacity   = 4

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  tags = local.tags
}
```

## ขั้นตอนที่ 3286: Automated Rollback Strategies

การทำ automated rollback เมื่อ deployment ล้มเหลวหรือ metrics แย่ลง

### ArgoCD Rollback

```bash
# ดู history
argocd app history shophub-user-service

# Rollback ไปยัง revision ก่อนหน้า
argocd app rollback shophub-user-service 5

# หรือ sync ไปยัง specific commit
argocd app set shophub-user-service --revision abc123
argocd app sync shophub-user-service
```

### Automated Rollback ด้วย GitHub Actions

```yaml
# .github/workflows/rollback.yml
name: Automated Rollback

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to rollback'
        required: true
        type: choice
        options:
          - user-service
          - product-service
          - order-service
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      revision:
        description: 'Git revision to rollback to (leave empty for previous)'
        required: false

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: myorg/shophub-gitops
          token: ${{ secrets.GITOPS_TOKEN }}
          fetch-depth: 10

      - name: Get previous image tag
        id: get-tag
        run: |
          if [ -n "${{ github.event.inputs.revision }}" ]; then
            # ใช้ revision ที่ระบุ
            git checkout ${{ github.event.inputs.revision }}
          else
            # ย้อนกลับ 1 commit
            git checkout HEAD~1
          fi
          
          PREV_TAG=$(cd kubernetes/${{ github.event.inputs.service }}/overlays/${{ github.event.inputs.environment }} \
            && kustomize build | grep 'image:' | awk '{print $2}')
          echo "previous_tag=$PREV_TAG" >> $GITHUB_OUTPUT

      - name: Apply rollback
        run: |
          git checkout main
          cd kubernetes/${{ github.event.inputs.service }}/overlays/${{ github.event.inputs.environment }}
          kustomize edit set image ${{ steps.get-tag.outputs.previous_tag }}
          git config user.email "ci@shophub.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "rollback: ${{ github.event.inputs.service }} in ${{ github.event.inputs.environment }}"
          git push

      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🔄 Rollback triggered for ${{ github.event.inputs.service }} in ${{ github.event.inputs.environment }} by ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Canary Deployment ด้วย Argo Rollouts

```yaml
# kubernetes/user-service/rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: user-service
spec:
  replicas: 10
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: myregistry.azurecr.io/shophub/user-service:latest
          ports:
            - containerPort: 8080
  strategy:
    canary:
      steps:
        - setWeight: 10    # ส่ง 10% traffic ไปยัง new version
        - pause:
            duration: 5m   # รอ 5 นาที
        - analysis:        # ตรวจสอบ metrics
            templates:
              - templateName: success-rate
        - setWeight: 30
        - pause:
            duration: 5m
        - setWeight: 60
        - pause:
            duration: 5m
        - setWeight: 100
      canaryService: user-service-canary
      stableService: user-service-stable
      trafficRouting:
        nginx:
          stableIngress: user-service-ingress
      analysis:
        successCondition: "result[0] >= 0.95"
        failureLimit: 3
```

```yaml
# AnalysisTemplate สำหรับตรวจสอบ success rate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_server_requests_seconds_count{
              service="user-service",
              status!~"5.."
            }[2m])) 
            / 
            sum(rate(http_server_requests_seconds_count{
              service="user-service"
            }[2m]))
    - name: avg-response-time
      interval: 1m
      count: 5
      successCondition: result[0] < 0.5
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.95,
              sum(rate(http_server_requests_seconds_bucket{
                service="user-service"
              }[2m])) by (le))
```

## ขั้นตอนที่ 3287: Dockerfile Best Practices สำหรับ Spring Boot

```dockerfile
# Multi-stage build สำหรับ Spring Boot
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Cache dependencies แยกจาก source code
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -q

# Build application
COPY src ./src
RUN ./mvnw package -DskipTests -q

# Extract layers สำหรับ better caching
RUN java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: สร้าง non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy layers ตามลำดับ frequency of change
COPY --from=build /app/dependencies/ ./
COPY --from=build /app/spring-boot-loader/ ./
COPY --from=build /app/snapshot-dependencies/ ./
COPY --from=build /app/application/ ./

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

# JVM flags สำหรับ container
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

## ขั้นตอนที่ 3288: Observability Stack

```yaml
# kubernetes/monitoring/prometheus-values.yaml
# Prometheus + Grafana + AlertManager
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    retention: 15d
    retentionSize: "50GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

    additionalScrapeConfigs:
      - job_name: 'spring-boot-services'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)

alertmanager:
  config:
    global:
      slack_api_url: '${SLACK_WEBHOOK_URL}'
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: slack-notifications
      routes:
        - match:
            severity: critical
          receiver: pagerduty-critical
    receivers:
      - name: slack-notifications
        slack_configs:
          - channel: '#alerts'
            text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
      - name: pagerduty-critical
        pagerduty_configs:
          - routing_key: '${PAGERDUTY_KEY}'
```

```java
// Prometheus Metrics ใน Spring Boot
@Configuration
public class MetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags(
            @Value("${spring.application.name}") String applicationName) {
        return registry -> registry.config()
                .commonTags("application", applicationName,
                           "environment", System.getenv("ENVIRONMENT"));
    }
}

// Custom metrics ใน service
@Service
@RequiredArgsConstructor
public class OrderMetricsService {

    private final MeterRegistry meterRegistry;

    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;

    @PostConstruct
    public void initMetrics() {
        Counter.builder("orders.created")
                .description("Total orders created")
                .tag("status", "success")
                .register(meterRegistry);
    }

    public void recordOrderCreated(String category) {
        meterRegistry.counter("orders.created", "category", category).increment();
    }

    public void recordOrderProcessingTime(long durationMs) {
        meterRegistry.timer("orders.processing.time")
                .record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

## ขั้นตอนที่ 3289-3320: Pipeline Best Practices

### Security Scanning ใน Pipeline

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: '0 2 * * 1'  # ทุกวันจันทร์ 2am

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'ShopHub'
          path: '.'
          format: 'HTML'
          args: >
            --enableRetired
            --failOnCVSS 7
            --suppression suppression.xml

      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: dependency-check-report
          path: ${{github.workspace}}/reports

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image for scanning
        run: docker build -t test-image:latest ./user-service

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'test-image:latest'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: java
          
      - name: Build
        run: mvn compile -DskipTests

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
```

### Deployment Verification

```java
// Smoke test หลัง deployment
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@TestPropertySource(properties = {
    "test.base-url=${TEST_BASE_URL:http://localhost:8080}"
})
class SmokeTest {

    @Value("${test.base-url}")
    private String baseUrl;

    private final RestTemplate restTemplate = new RestTemplate();

    @Test
    void healthEndpointShouldBeUp() {
        ResponseEntity<Map> response = restTemplate
                .getForEntity(baseUrl + "/actuator/health", Map.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().get("status")).isEqualTo("UP");
    }

    @Test
    void apiShouldRespond() {
        ResponseEntity<String> response = restTemplate
                .getForEntity(baseUrl + "/actuator/info", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

*[← Part 91: Microservices Project](./part-91-microservices-project.md) | [Part 93: Cost Optimization →](./part-93-cost-optimization.md)*

# Part 43: Infrastructure as Code - Terraform & Helm
## ขั้นตอนที่ 1316-1355

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 6-7 ชั่วโมง  
> **เป้าหมาย:** Manage infrastructure declaratively

---

## ขั้นตอนที่ 1316: Infrastructure as Code Concepts

```
Infrastructure as Code (IaC):
  - Manage servers, databases, networks as code
  - Version control for infrastructure
  - Reproducible environments
  - Audit trail of changes

Tools:
  Terraform  = Cloud-agnostic (AWS, GCP, Azure, K8s)
  Pulumi     = IaC with real programming languages
  Ansible    = Configuration management
  Helm       = Kubernetes package manager
  Kustomize  = Kubernetes YAML customization

Benefits:
  ✅ Reproducible (same infra every time)
  ✅ Version controlled
  ✅ Code review for infra changes
  ✅ Disaster recovery
  ✅ Multi-environment management
```

---

## ขั้นตอนที่ 1317: Terraform for AWS

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "production/terraform.tfstate"
    region = "ap-southeast-1"
    
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "myapp"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# Variables
variable "aws_region" {
  default = "ap-southeast-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "app_name" {
  default = "myapp"
}

# VPC
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${var.app_name}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = var.environment != "production"
  
  enable_dns_hostnames = true
  enable_dns_support   = true
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "${var.app_name}-${var.environment}"
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  cluster_endpoint_public_access = true
  
  # Node Groups
  eks_managed_node_groups = {
    main = {
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3
      
      labels = {
        Environment = var.environment
      }
    }
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "postgres" {
  identifier = "${var.app_name}-${var.environment}-db"
  
  engine         = "postgres"
  engine_version = "16"
  instance_class = var.environment == "production" ? "db.r6g.large" : "db.t3.micro"
  
  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true
  
  db_name  = var.app_name
  username = "postgres"
  password = random_password.db.result
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = var.environment == "production" ? 7 : 1
  deletion_protection     = var.environment == "production"
  
  performance_insights_enabled = var.environment == "production"
  
  skip_final_snapshot = var.environment != "production"
}

resource "random_password" "db" {
  length  = 32
  special = true
}

# ElastiCache Redis
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "${var.app_name}-${var.environment}-redis"
  description          = "Redis cluster for ${var.app_name}"
  
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = var.environment == "production" ? "cache.r6g.large" : "cache.t3.micro"
  
  num_cache_clusters   = var.environment == "production" ? 2 : 1
  
  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}

# Outputs
output "eks_cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "rds_endpoint" {
  value     = aws_db_instance.postgres.endpoint
  sensitive = true
}

output "redis_endpoint" {
  value     = aws_elasticache_replication_group.redis.primary_endpoint_address
  sensitive = true
}
```

---

## ขั้นตอนที่ 1318: Helm Charts

```yaml
# helm/myapp/Chart.yaml
apiVersion: v2
name: myapp
description: My Spring Boot Application
type: application
version: 0.1.0
appVersion: "1.0.0"

# helm/myapp/values.yaml
replicaCount: 3

image:
  repository: ghcr.io/myorg/myapp
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - api.myapp.com

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

config:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"

secrets:
  SPRING_DATASOURCE_URL: ""
  SPRING_DATASOURCE_USERNAME: ""
  SPRING_DATASOURCE_PASSWORD: ""
  JWT_SECRET: ""

postgresql:
  enabled: true
  auth:
    database: myapp
  primary:
    persistence:
      size: 20Gi

redis:
  enabled: true
  auth:
    enabled: false
```

```yaml
# helm/myapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}-config
            - secretRef:
                name: {{ include "myapp.fullname" . }}-secrets
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 60
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

## ขั้นตอนที่ 1319: Terraform Commands

```bash
# Initialize
terraform init

# Preview changes
terraform plan -var="environment=production" -out=tfplan

# Apply
terraform apply tfplan

# Destroy (careful!)
terraform destroy -var="environment=staging"

# State management
terraform state list
terraform state show aws_db_instance.postgres

# Import existing resource
terraform import aws_db_instance.postgres myapp-prod-db

# Workspace for multiple environments
terraform workspace new staging
terraform workspace new production
terraform workspace select production
```

---

## ขั้นตอนที่ 1320-1355: Helm Commands

```bash
# Add chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install
helm install myapp ./helm/myapp \
  --namespace myapp \
  --create-namespace \
  --values helm/myapp/values.production.yaml \
  --set image.tag="sha-abc123"

# Upgrade
helm upgrade myapp ./helm/myapp \
  --namespace myapp \
  --set image.tag="sha-def456"

# Rollback
helm rollback myapp 1 --namespace myapp

# List releases
helm list --namespace myapp

# Get values
helm get values myapp --namespace myapp

# Dry run
helm upgrade myapp ./helm/myapp \
  --namespace myapp \
  --dry-run --debug

# Template rendering (check output)
helm template myapp ./helm/myapp \
  --values helm/myapp/values.production.yaml
```

---

*[← Part 42: CI/CD](./part-42-cicd.md) | [Part 44: Complete E-Commerce Project →](./part-44-complete-project.md)*

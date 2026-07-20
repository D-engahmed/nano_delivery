# SmartFleet Deployment Guide
## Production Deployment on AWS EKS

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Local Development Setup](#2-local-development-setup)
3. [AWS Infrastructure (Terraform)](#3-aws-infrastructure-terraform)
4. [Kubernetes Deployment](#4-kubernetes-deployment)
5. [CI/CD Pipeline](#5-cicd-pipeline)
6. [Monitoring & Alerting](#6-monitoring--alerting)
7. [Disaster Recovery](#7-disaster-recovery)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Prerequisites

### Required Tools

| Tool | Version | Purpose | Install Command |
|------|---------|---------|-----------------|
| Docker | 24.0+ | Containerization | [docker.com](https://docker.com) |
| kubectl | 1.29+ | Kubernetes CLI | `brew install kubectl` |
| Terraform | 1.7+ | IaC | `brew install terraform` |
| AWS CLI | 2.15+ | AWS management | `brew install awscli` |
| Helm | 3.14+ | K8s package manager | `brew install helm` |
| eksctl | 0.170+ | EKS management | `brew install eksctl` |
| k9s | 0.31+ | K8s TUI | `brew install k9s` |

### AWS Account Setup

```bash
# Configure AWS CLI
aws configure
# Enter: AWS Access Key ID, Secret Access Key, region (ap-southeast-1), output (json)

# Verify access
aws sts get-caller-identity

# Create S3 bucket for Terraform state (one-time)
aws s3 mb s3://smartfleet-terraform-state-prod --region ap-southeast-1
aws s3api put-bucket-versioning     --bucket smartfleet-terraform-state-prod     --versioning-configuration Status=Enabled

# Create DynamoDB table for Terraform state locking
aws dynamodb create-table     --table-name smartfleet-terraform-locks     --attribute-definitions AttributeName=LockID,AttributeType=S     --key-schema AttributeName=LockID,KeyType=HASH     --billing-mode PAY_PER_REQUEST     --region ap-southeast-1
```

### Environment Variables

```bash
# Create .env file
cat > .env << 'EOF'
# AWS
AWS_REGION=ap-southeast-1
AWS_PROFILE=smartfleet-prod

# Database
DB_USERNAME=smartfleet_admin
DB_PASSWORD=$(openssl rand -base64 32)
DB_NAME=smartfleet_db

# Redis
REDIS_PASSWORD=$(openssl rand -base64 32)

# JWT
JWT_SECRET=$(openssl rand -base64 64)

# Application
ENVIRONMENT=production
LOG_LEVEL=INFO

# Monitoring
GRAFANA_PASSWORD=$(openssl rand -base64 16)
EOF

source .env
```

---

## 2. Local Development Setup

### Quick Start (Docker Compose)

```bash
# Clone repository
git clone https://github.com/yourusername/smartfleet.git
cd smartfleet

# Start all services
docker-compose up -d

# Verify services are running
docker-compose ps

# View logs
docker-compose logs -f order-service

# Run database migrations
docker-compose exec order-service alembic upgrade head

# Seed test data
docker-compose exec order-service python -m scripts.seed_data

# Access services
# API Gateway: http://localhost:80
# Order Service: http://localhost:8001/docs
# Fleet Service: http://localhost:8002/docs
# Grafana: http://localhost:3000 (admin/admin)
# Prometheus: http://localhost:9090
```

### Docker Compose Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Network: smartfleet                 │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Nginx   │  │  Order   │  │  Fleet   │  │   UTM    │  │
│  │  :80     │  │  :8001   │  │  :8002   │  │  :8003   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │             │             │             │        │
│       └─────────────┴─────────────┴─────────────┘        │
│                         │                                   │
│  ┌──────────┐  ┌───────┴──────┐  ┌──────────────────┐     │
│  │PostgreSQL │  │    Redis     │  │    RabbitMQ      │     │
│  │  :5432   │  │   :6379      │  │    :5672         │     │
│  └──────────┘  └──────────────┘  └──────────────────┘     │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │Prometheus│  │ Grafana  │  │  ML      │                │
│  │  :9090   │  │  :3000   │  │  :8004   │                │
│  └──────────┘  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. AWS Infrastructure (Terraform)

### Directory Structure

```
infrastructure/terraform/
├── main.tf              # Main infrastructure
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── providers.tf         # Provider configuration
├── backend.tf           # Remote state configuration
├── vpc.tf               # Networking
├── eks.tf               # Kubernetes cluster
├── rds.tf               # Database
├── elasticache.tf       # Redis cache
├── s3.tf                # Object storage
├── alb.tf               # Load balancer
├── iam.tf               # Permissions
├── security.tf          # Security groups
├── monitoring.tf        # CloudWatch
└── environments/
    ├── prod.tfvars      # Production values
    └── dev.tfvars       # Development values
```

### 3.1 Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "smartfleet-terraform-state-prod"
    key            = "infrastructure/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "smartfleet-terraform-locks"
  }
}
```

### 3.2 Provider & Versions

```hcl
# providers.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "SmartFleet"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  token                  = data.aws_eks_cluster_auth.main.token
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    token                  = data.aws_eks_cluster_auth.main.token
  }
}
```

### 3.3 VPC & Networking

```hcl
# vpc.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "smartfleet-${var.environment}"
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  # Enable for private subnets (needed for RDS)
  create_database_subnet_group = true
  database_subnets            = var.database_subnet_cidrs

  # NAT Gateway for outbound internet from private subnets
  enable_nat_gateway     = true
  single_nat_gateway     = var.environment == "dev"
  one_nat_gateway_per_az = var.environment == "prod"

  # DNS
  enable_dns_hostnames = true
  enable_dns_support   = true

  # Tags for Kubernetes integration
  public_subnet_tags = {
    "kubernetes.io/role/elb"                    = "1"
    "kubernetes.io/cluster/smartfleet-${var.environment}" = "shared"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"           = "1"
    "kubernetes.io/cluster/smartfleet-${var.environment}" = "shared"
  }

  tags = {
    Name = "smartfleet-${var.environment}"
  }
}
```

### 3.4 EKS Cluster

```hcl
# eks.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "smartfleet-${var.environment}"
  cluster_version = "1.29"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  control_plane_subnet_ids       = module.vpc.private_subnets

  # Public endpoint for kubectl access (restrict in production)
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # Enable logging
  cluster_enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  # EKS Managed Node Groups
  eks_managed_node_groups = {
    general = {
      name = "general-workloads"

      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"

      min_size     = 2
      max_size     = 10
      desired_size = 3

      labels = {
        workload = "general"
      }

      taints = []

      block_device_mappings = {
        xvda = {
          device_name = "/dev/xvda"
          ebs = {
            volume_size           = 50
            volume_type           = "gp3"
            iops                  = 3000
            encrypted             = true
            delete_on_termination = true
          }
        }
      }
    }

    ml = {
      name = "ml-workloads"

      instance_types = ["g4dn.xlarge"]  # GPU instances for ML inference
      capacity_type  = "ON_DEMAND"

      min_size     = 0
      max_size     = 5
      desired_size = 1

      labels = {
        workload = "ml"
        gpu      = "true"
      }

      taints = [{
        key    = "nvidia.com/gpu"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }

    spot = {
      name = "spot-workloads"

      instance_types = ["t3.medium", "t3a.medium", "t3.large"]
      capacity_type  = "SPOT"

      min_size     = 1
      max_size     = 20
      desired_size = 3

      labels = {
        workload = "spot"
      }

      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }

  # Fargate for serverless workloads (optional)
  fargate_profiles = {
    default = {
      name = "default"
      selectors = [
        { namespace = "kube-system" },
        { namespace = "smartfleet" }
      ]
    }
  }

  # IRSA (IAM Roles for Service Accounts)
  enable_irsa = true

  tags = {
    Name = "smartfleet-${var.environment}"
  }
}

# Install NVIDIA device plugin for GPU nodes (ML workloads)
resource "helm_release" "nvidia_device_plugin" {
  name       = "nvidia-device-plugin"
  repository = "https://nvidia.github.io/k8s-device-plugin"
  chart      = "nvidia-device-plugin"
  namespace  = "kube-system"
  version    = "0.14.0"

  set {
    name  = "nodeSelector.workload"
    value = "ml"
  }

  depends_on = [module.eks]
}
```

### 3.5 RDS PostgreSQL

```hcl
# rds.tf
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "smartfleet-${var.environment}"

  engine               = "postgres"
  engine_version       = "15.4"
  family               = "postgres15"
  major_engine_version = "15"
  instance_class       = var.db_instance_class

  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = var.db_name
  username = var.db_username
  port     = 5432

  multi_az               = var.environment == "prod"
  db_subnet_group_name   = module.vpc.database_subnet_group_name
  vpc_security_group_ids = [aws_security_group.database.id]

  # Backup
  backup_retention_period = var.environment == "prod" ? 30 : 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  # Performance Insights
  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  # Enhanced monitoring
  monitoring_interval    = 60
  monitoring_role_name   = "smartfleet-rds-monitoring"
  create_monitoring_role = true

  # Deletion protection (disable in dev for easy cleanup)
  deletion_protection = var.environment == "prod"
  skip_final_snapshot = var.environment != "prod"

  tags = {
    Name = "smartfleet-db-${var.environment}"
  }
}

# Security group for RDS
resource "aws_security_group" "database" {
  name_prefix = "smartfleet-db-${var.environment}-"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [module.eks.cluster_security_group_id]
    description     = "EKS cluster access"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "smartfleet-db-${var.environment}"
  }
}

# Store DB credentials in Secrets Manager
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "smartfleet/${var.environment}/database"
  description             = "Database credentials for SmartFleet"
  recovery_window_in_days = var.environment == "prod" ? 30 : 0
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
    host     = module.rds.db_instance_address
    port     = 5432
    dbname   = var.db_name
    url      = "postgresql://${var.db_username}:${var.db_password}@${module.rds.db_instance_address}:5432/${var.db_name}"
  })
}
```

### 3.6 ElastiCache Redis

```hcl
# elasticache.tf
module "redis" {
  source  = "terraform-aws-modules/elasticache/aws"
  version = "~> 1.0"

  cluster_id = "smartfleet-${var.environment}"

  engine               = "redis"
  engine_version       = "7.1"
  node_type            = var.redis_node_type
  num_cache_nodes      = var.environment == "prod" ? 2 : 1
  parameter_group_name = "default.redis7"
  port                 = 6379

  subnet_group_name  = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_password

  snapshot_retention_limit = var.environment == "prod" ? 7 : 1
  snapshot_window         = "05:00-06:00"

  tags = {
    Name = "smartfleet-redis-${var.environment}"
  }
}

resource "aws_elasticache_subnet_group" "redis" {
  name       = "smartfleet-redis-${var.environment}"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_security_group" "redis" {
  name_prefix = "smartfleet-redis-${var.environment}-"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [module.eks.cluster_security_group_id]
  }

  tags = {
    Name = "smartfleet-redis-${var.environment}"
  }
}
```

### 3.7 Application Load Balancer

```hcl
# alb.tf
module "alb" {
  source  = "terraform-aws-modules/alb/aws"
  version = "~> 9.0"

  name = "smartfleet-${var.environment}"

  load_balancer_type = "application"
  vpc_id             = module.vpc.vpc_id
  subnets            = module.vpc.public_subnets
  security_groups    = [aws_security_group.alb.id]

  # Access logs
  access_logs = {
    bucket  = aws_s3_bucket.logs.id
    enabled = true
  }

  # Listeners
  listeners = {
    http = {
      port     = 80
      protocol = "HTTP"
      redirect = {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }

    https = {
      port            = 443
      protocol        = "HTTPS"
      certificate_arn = aws_acm_certificate.main.arn

      fixed_response = {
        content_type = "text/plain"
        message_body = "OK"
        status_code  = "200"
      }
    }
  }

  # Target groups (will be managed by Kubernetes Ingress)
  target_groups = {}

  tags = {
    Name = "smartfleet-alb-${var.environment}"
  }
}

# ACM Certificate
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = ["*.${var.domain_name}"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}
```

### 3.8 S3 Buckets

```hcl
# s3.tf
resource "aws_s3_bucket" "storage" {
  bucket = "smartfleet-storage-${var.environment}-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "storage" {
  bucket = aws_s3_bucket.storage.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "storage" {
  bucket = aws_s3_bucket.storage.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "storage" {
  bucket = aws_s3_bucket.storage.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IAM policy for pods to access S3
resource "aws_iam_policy" "s3_access" {
  name        = "smartfleet-s3-${var.environment}"
  description = "Allow SmartFleet pods to access S3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.storage.arn,
          "${aws_s3_bucket.storage.arn}/*"
        ]
      }
    ]
  })
}

# IRSA for S3 access
module "s3_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.0"

  role_name = "smartfleet-s3-${var.environment}"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["smartfleet:order-service", "smartfleet:fleet-service"]
    }
  }

  role_policy_arns = {
    s3 = aws_iam_policy.s3_access.arn
  }
}
```

### 3.9 Apply Terraform

```bash
# Initialize Terraform
cd infrastructure/terraform
terraform init

# Plan changes
terraform plan -var-file="environments/prod.tfvars" -out=tfplan

# Review plan
terraform show tfplan

# Apply
terraform apply tfplan

# Verify outputs
terraform output

# Configure kubectl
aws eks update-kubeconfig     --region ap-southeast-1     --name smartfleet-prod

# Verify cluster access
kubectl get nodes
kubectl get pods -n kube-system
```

---

## 4. Kubernetes Deployment

### 4.1 Namespace & Config

```yaml
# infrastructure/kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: smartfleet
  labels:
    istio-injection: enabled  # If using Istio service mesh
---
# infrastructure/kubernetes/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: smartfleet-config
  namespace: smartfleet
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  REDIS_URL: "redis://smartfleet-redis.prod.cache.amazonaws.com:6379/0"
  RABBITMQ_URL: "amqp://user:pass@rabbitmq:5672/"
  ML_SERVICE_URL: "http://ml-route-optimizer:8000"
  AWS_REGION: "ap-southeast-1"
  S3_BUCKET: "smartfleet-storage-prod-123456789"
---
# infrastructure/kubernetes/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: smartfleet
type: Opaque
stringData:
  url: "postgresql://user:pass@smartfleet-db.xxx.ap-southeast-1.rds.amazonaws.com:5432/smartfleet_db"
---
apiVersion: v1
kind: Secret
metadata:
  name: auth-secret
  namespace: smartfleet
type: Opaque
stringData:
  jwt_secret: "your-256-bit-secret-here"
  jwt_refresh_secret: "your-256-bit-refresh-secret"
```

### 4.2 Order Service Deployment

```yaml
# infrastructure/kubernetes/deployments/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: smartfleet
  labels:
    app: order-service
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: smartfleet-s3  # IRSA for S3 access

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - order-service
              topologyKey: kubernetes.io/hostname

      containers:
      - name: order-service
        image: "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/smartfleet-order-service:v1.0.0"
        imagePullPolicy: Always

        ports:
        - name: http
          containerPort: 8000
          protocol: TCP

        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: smartfleet-config
              key: REDIS_URL
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-secret
              key: jwt_secret
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: smartfleet-config
              key: ENVIRONMENT

        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 12  # 60 seconds total

        volumeMounts:
        - name: tmp
          mountPath: /tmp

      volumes:
      - name: tmp
        emptyDir: {}

      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: smartfleet
  labels:
    app: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: smartfleet
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
---
# Pod Disruption Budget (ensure minimum availability during updates)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: smartfleet
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

### 4.3 Ingress Configuration

```yaml
# infrastructure/kubernetes/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: smartfleet-ingress
  namespace: smartfleet
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-southeast-1:123456789:certificate/xxx
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=smartfleet-logs,
      access_logs.s3.prefix=alb
spec:
  ingressClassName: alb
  rules:
  - host: api.smartfleet.io
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /fleet
        pathType: Prefix
        backend:
          service:
            name: fleet-service
            port:
              number: 80
      - path: /utm
        pathType: Prefix
        backend:
          service:
            name: utm-service
            port:
              number: 80
      - path: /analytics
        pathType: Prefix
        backend:
          service:
            name: analytics-service
            port:
              number: 80
      - path: /ml
        pathType: Prefix
        backend:
          service:
            name: ml-route-optimizer
            port:
              number: 80
```

### 4.4 Deploy to Kubernetes

```bash
# Apply all configurations
kubectl apply -f infrastructure/kubernetes/namespace.yaml
kubectl apply -f infrastructure/kubernetes/configmap.yaml
kubectl apply -f infrastructure/kubernetes/secrets.yaml
kubectl apply -f infrastructure/kubernetes/deployments/
kubectl apply -f infrastructure/kubernetes/services/
kubectl apply -f infrastructure/kubernetes/ingress.yaml

# Verify deployments
kubectl get deployments -n smartfleet
kubectl get pods -n smartfleet
kubectl get svc -n smartfleet
kubectl get ingress -n smartfleet

# Check pod logs
kubectl logs -n smartfleet deployment/order-service --tail=100

# Port-forward for local testing
kubectl port-forward -n smartfleet svc/order-service 8001:80

# Scale manually if needed
kubectl scale deployment order-service -n smartfleet --replicas=5
```

---

## 5. CI/CD Pipeline

### 5.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: ap-southeast-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com
  EKS_CLUSTER: smartfleet-prod

jobs:
  # Stage 1: Lint & Test
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [order-service, fleet-service, utm-service]

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Cache pip dependencies
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('services/${{ matrix.service }}/requirements.txt') }}

    - name: Install dependencies
      run: |
        cd services/${{ matrix.service }}
        pip install -r requirements.txt
        pip install pytest pytest-cov pytest-asyncio black flake8 mypy bandit

    - name: Lint with flake8
      run: |
        cd services/${{ matrix.service }}
        flake8 app --max-line-length=100 --exclude=__pycache__ --statistics

    - name: Format check with black
      run: |
        cd services/${{ matrix.service }}
        black --check app

    - name: Type check with mypy
      run: |
        cd services/${{ matrix.service }}
        mypy app --ignore-missing-imports --strict

    - name: Security scan with bandit
      run: |
        cd services/${{ matrix.service }}
        bandit -r app -f json -o bandit-report.json || true

    - name: Run tests with coverage
      run: |
        cd services/${{ matrix.service }}
        pytest tests/ -v --cov=app --cov-report=xml --cov-report=html

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: services/${{ matrix.service }}/coverage.xml
        flags: ${{ matrix.service }}
        fail_ci_if_error: false

  # Stage 2: Build & Push Docker Images
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    strategy:
      matrix:
        service: [order-service, fleet-service, utm-service, ml-route-optimizer]

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: services/${{ matrix.service }}
        file: services/${{ matrix.service }}/Dockerfile
        push: true
        tags: |
          ${{ env.ECR_REGISTRY }}/smartfleet-${{ matrix.service }}:${{ github.sha }}
          ${{ env.ECR_REGISTRY }}/smartfleet-${{ matrix.service }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Scan image for vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.ECR_REGISTRY }}/smartfleet-${{ matrix.service }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
      if: always()

  # Stage 3: Deploy to Kubernetes
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig           --region ${{ env.AWS_REGION }}           --name ${{ env.EKS_CLUSTER }}

    - name: Deploy to Kubernetes
      run: |
        # Update image tags
        sed -i 's|image: .*|image: ${{ env.ECR_REGISTRY }}/smartfleet-order-service:${{ github.sha }}|'           infrastructure/kubernetes/deployments/order-service.yaml
        sed -i 's|image: .*|image: ${{ env.ECR_REGISTRY }}/smartfleet-fleet-service:${{ github.sha }}|'           infrastructure/kubernetes/deployments/fleet-service.yaml
        sed -i 's|image: .*|image: ${{ env.ECR_REGISTRY }}/smartfleet-utm-service:${{ github.sha }}|'           infrastructure/kubernetes/deployments/utm-service.yaml

        # Apply configurations
        kubectl apply -k infrastructure/kubernetes/

        # Wait for rollout
        kubectl rollout status deployment/order-service -n smartfleet --timeout=300s
        kubectl rollout status deployment/fleet-service -n smartfleet --timeout=300s
        kubectl rollout status deployment/utm-service -n smartfleet --timeout=300s

    - name: Run smoke tests
      run: |
        kubectl run smoke-test --rm -i --restart=Never           --image=curlimages/curl:latest           -- curl -f http://order-service.smartfleet.svc.cluster.local/health

        kubectl run smoke-test --rm -i --restart=Never           --image=curlimages/curl:latest           -- curl -f http://fleet-service.smartfleet.svc.cluster.local/health

    - name: Notify Slack on success
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: "Deployment to production successful! :rocket:"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      if: success()

  # Stage 4: Rollback on failure
  rollback:
    needs: deploy
    runs-on: ubuntu-latest
    if: failure()

    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig           --region ${{ env.AWS_REGION }}           --name ${{ env.EKS_CLUSTER }}

    - name: Rollback deployment
      run: |
        kubectl rollout undo deployment/order-service -n smartfleet
        kubectl rollout undo deployment/fleet-service -n smartfleet
        kubectl rollout undo deployment/utm-service -n smartfleet

    - name: Notify Slack on failure
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: "Deployment failed! Rolling back... :warning:"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 5.2 GitOps with ArgoCD (Optional Advanced)

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: smartfleet
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourusername/smartfleet.git
    targetRevision: main
    path: infrastructure/kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: smartfleet
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 6. Monitoring & Alerting

### 6.1 Prometheus Configuration

```yaml
# monitoring/prometheus/prometheus-config.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: smartfleet-prod
    replica: '{{.ExternalURL}}'

alerting:
  alertmanagers:
  - static_configs:
    - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/rules/*.yaml

scrape_configs:
  # Kubernetes API Server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
    - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https

  # Kubernetes Nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
    - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)

  # Kubernetes Pods (auto-discovery)
  - job_name: 'kubernetes-pods'
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
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: kubernetes_pod_name

  # Application-specific metrics
  - job_name: 'smartfleet-services'
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
        - smartfleet
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_label_app]
      action: keep
      regex: (order-service|fleet-service|utm-service|ml-route-optimizer)
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: (.+)
```

### 6.2 Prometheus Rules (Alerts)

```yaml
# monitoring/prometheus/rules/smartfleet-alerts.yaml
groups:
- name: smartfleet-service-alerts
  rules:
  # High error rate
  - alert: HighErrorRate
    expr: |
      (
        sum(rate(http_requests_total{status=~"5.."}[5m])) 
        / 
        sum(rate(http_requests_total[5m]))
      ) > 0.05
    for: 5m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"

  # High latency
  - alert: HighLatency
    expr: |
      histogram_quantile(0.95, 
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
      ) > 0.5
    for: 10m
    labels:
      severity: warning
      team: backend
    annotations:
      summary: "High latency detected"
      description: "p95 latency is {{ $value }}s for {{ $labels.service }}"

  # Service down
  - alert: ServiceDown
    expr: up{job="smartfleet-services"} == 0
    for: 1m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "Service is down"
      description: "{{ $labels.instance }} has been down for more than 1 minute"

  # Database connections high
  - alert: DatabaseConnectionsHigh
    expr: |
      pg_stat_activity_count / pg_settings_max_connections * 100 > 80
    for: 5m
    labels:
      severity: warning
      team: backend
    annotations:
      summary: "Database connection pool high"
      description: "{{ $value }}% of connections in use"

  # Low battery on vehicles
  - alert: VehicleLowBattery
    expr: |
      vehicle_battery_level < 20
    for: 1m
    labels:
      severity: warning
      team: operations
    annotations:
      summary: "Vehicle low battery"
      description: "Vehicle {{ $labels.vehicle_id }} battery at {{ $value }}%"

  # Delivery delay
  - alert: DeliveryDelayed
    expr: |
      (
        time() - order_scheduled_delivery_time 
      ) > 600 AND order_status == "in_transit"
    for: 5m
    labels:
      severity: warning
      team: operations
    annotations:
      summary: "Delivery delayed"
      description: "Order {{ $labels.order_id }} is 10+ minutes late"

- name: infrastructure-alerts
  rules:
  # Node CPU high
  - alert: NodeCPUHigh
    expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"

  # Node memory high
  - alert: NodeMemoryHigh
    expr: |
      (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) 
      / node_memory_MemTotal_bytes * 100 > 85
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"

  # Disk space low
  - alert: DiskSpaceLow
    expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes * 100) < 10
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Low disk space on {{ $labels.instance }}"
```

### 6.3 Grafana Dashboard

```json
// monitoring/grafana/dashboards/smartfleet-overview.json
{
  "dashboard": {
    "title": "SmartFleet Overview",
    "tags": ["smartfleet", "production"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Orders Per Minute",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(orders_created_total[1m]))",
            "legendFormat": "Orders/min"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Active Vehicles",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(vehicle_status{status="busy"}) by (vehicle_type)",
            "legendFormat": "{{ vehicle_type }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "id": 3,
        "title": "API Latency (p95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "{{ service }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8}
      },
      {
        "id": 4,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))",
            "legendFormat": "Error %"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8}
      },
      {
        "id": 5,
        "title": "Delivery Success Rate",
        "type": "gauge",
        "targets": [
          {
            "expr": "sum(orders_status_total{status="delivered"}) / sum(orders_status_total)",
            "legendFormat": "Success Rate"
          }
        ],
        "gridPos": {"h": 8, "w": 8, "x": 0, "y": 16}
      },
      {
        "id": 6,
        "title": "Average Delivery Time",
        "type": "stat",
        "targets": [
          {
            "expr": "avg(order_delivery_duration_seconds)",
            "legendFormat": "Avg Duration"
          }
        ],
        "gridPos": {"h": 8, "w": 8, "x": 8, "y": 16}
      },
      {
        "id": 7,
        "title": "Fleet Battery Levels",
        "type": "graph",
        "targets": [
          {
            "expr": "avg(vehicle_battery_level) by (vehicle_type)",
            "legendFormat": "{{ vehicle_type }}"
          }
        ],
        "gridPos": {"h": 8, "w": 8, "x": 16, "y": 16}
      }
    ]
  }
}
```

---

## 7. Disaster Recovery

### 7.1 Backup Strategy

| Resource | Method | Frequency | Retention |
|----------|--------|-----------|-----------|
| PostgreSQL | Automated RDS snapshots | Daily | 30 days |
| PostgreSQL | Manual snapshots | Before deployments | 7 days |
| Redis | ElastiCache snapshots | Daily | 7 days |
| S3 Objects | Versioning + Cross-region replication | Continuous | 90 days |
| Kubernetes | Velero | Daily | 14 days |

### 7.2 RDS Point-in-Time Recovery

```bash
# Restore database to a specific point in time
aws rds restore-db-instance-to-point-in-time     --source-db-instance-identifier smartfleet-prod     --target-db-instance-identifier smartfleet-prod-restored     --restore-time 2026-07-20T10:00:00Z     --db-instance-class db.t3.medium
```

### 7.3 Velero (Kubernetes Backup)

```bash
# Install Velero
velero install     --provider aws     --bucket smartfleet-backups     --backup-location-config region=ap-southeast-1     --snapshot-location-config region=ap-southeast-1     --secret-file ./credentials-velero

# Create backup
velero backup create smartfleet-backup-$(date +%Y%m%d)     --include-namespaces smartfleet     --ttl 720h0m0s

# Restore from backup
velero restore create --from-backup smartfleet-backup-20260720
```

### 7.4 Multi-Region DR (Advanced)

```
Primary Region: ap-southeast-1 (Singapore)
  ├─ EKS Cluster (active)
  ├─ RDS Primary (Multi-AZ)
  ├─ ElastiCache Primary
  └─ ALB (active)

DR Region: ap-northeast-1 (Tokyo)
  ├─ EKS Cluster (standby, scaled to 0)
  ├─ RDS Read Replica (can be promoted)
  ├─ ElastiCache (standby)
  └─ ALB (standby)

S3: Cross-region replication to DR bucket
Route53: Health checks + failover routing
```

---

## 8. Troubleshooting

### 8.1 Common Issues

```bash
# Pods not starting
kubectl describe pod <pod-name> -n smartfleet
kubectl logs <pod-name> -n smartfleet --previous

# High memory usage
kubectl top pods -n smartfleet
kubectl top nodes

# Database connection issues
kubectl exec -it <pod-name> -n smartfleet -- python -c "import psycopg2; conn = psycopg2.connect('...')"

# Redis connectivity
kubectl exec -it <pod-name> -n smartfleet -- redis-cli -h <redis-host> ping

# Network issues between services
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash
# Then: curl http://order-service.smartfleet.svc.cluster.local/health

# HPA not scaling
kubectl describe hpa order-service-hpa -n smartfleet
kubectl get events -n smartfleet --field-selector reason=FailedScheduling
```

### 8.2 Performance Tuning

```bash
# Check slow queries in PostgreSQL
kubectl exec -it <postgres-pod> -- psql -U smartfleet_admin -d smartfleet_db -c "
SELECT query, calls, mean_time, total_time 
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;
"

# Analyze table statistics
kubectl exec -it <postgres-pod> -- psql -U smartfleet_admin -d smartfleet_db -c "
ANALYZE orders;
ANALYZE vehicle_telemetry;
"

# Check Redis memory usage
kubectl exec -it <redis-pod> -- redis-cli INFO memory

# Check for connection leaks
kubectl exec -it <postgres-pod> -- psql -U smartfleet_admin -d smartfleet_db -c "
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;
"
```

### 8.3 Emergency Procedures

```bash
# Scale all services to 0 (emergency stop)
kubectl scale deployment --all --replicas=0 -n smartfleet

# Rollback to previous version
kubectl rollout undo deployment/order-service -n smartfleet

# Restart all pods
kubectl rollout restart deployment -n smartfleet

# Evacuate node (for maintenance)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Force delete stuck pod
kubectl delete pod <pod-name> -n smartfleet --force --grace-period=0
```

---

## Cost Estimation (Monthly, AWS Singapore)

| Service | Instance | Monthly Cost |
|---------|----------|-------------|
| EKS Cluster | Control plane | $73 |
| EKS Nodes (3x t3.medium) | ON_DEMAND | $180 |
| EKS Nodes (1x g4dn.xlarge) | ON_DEMAND | $600 |
| RDS PostgreSQL | db.t3.medium, Multi-AZ | $250 |
| ElastiCache Redis | cache.t3.micro | $30 |
| ALB | LCU-based | $25 |
| S3 Storage | 100GB | $3 |
| Data Transfer | 500GB | $45 |
| CloudWatch Logs | 10GB/day | $50 |
| **Total** | | **~$1,256/month** |

**Cost Optimization:**
- Use Spot instances for non-critical workloads: **-40%**
- Reserved Instances for 1-year commitment: **-30%**
- Use Graviton (ARM) instances: **-20%**
- **Optimized total: ~$600-800/month**

# CI/CD & DevOps Strategy

## DevOps Philosophy

PhotoShare follows modern DevOps practices with emphasis on:
- **Infrastructure as Code**: All infrastructure defined in version control
- **Automated Testing**: Comprehensive test suite at all levels
- **Continuous Integration**: Automated builds and tests on every commit
- **Continuous Deployment**: Automated deployments with safety checks
- **Monitoring & Observability**: Built-in monitoring from day one

## CI/CD Pipeline Architecture

### Pipeline Overview
```mermaid
graph TB
    A[Developer Push] --> B[GitHub Repository]
    B --> C[GitHub Actions Trigger]
    C --> D[Code Quality Checks]
    D --> E[Unit Tests]
    E --> F[Integration Tests]
    F --> G[Security Scan]
    G --> H[Build Docker Images]
    H --> I[Push to ECR]
    I --> J[Deploy to Staging]
    J --> K[E2E Tests]
    K --> L[Performance Tests]
    L --> M[Manual Approval Gate]
    M --> N[Deploy to Production]
    N --> O[Health Checks]
    O --> P[Rollback on Failure]
```

### GitHub Actions Workflows

#### Main CI/CD Workflow
```yaml
name: PhotoShare CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: photoshare-api
  ECS_SERVICE: photoshare-service
  ECS_CLUSTER: photoshare-cluster

jobs:
  code-quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: ESLint
        run: npm run lint
        
      - name: Prettier
        run: npm run format:check
        
      - name: TypeScript Check
        run: npm run type-check
        
      - name: Dependency Audit
        run: npm audit --audit-level=moderate

  unit-tests:
    runs-on: ubuntu-latest
    needs: code-quality
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: photoshare_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run unit tests
        run: npm run test:unit
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/photoshare_test
          REDIS_URL: redis://localhost:6379
          
      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: photoshare_test
      redis:
        image: redis:7
      localstack:
        image: localstack/localstack:latest
        env:
          SERVICES: s3,sqs,sns,lambda
          
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/photoshare_test
          REDIS_URL: redis://localhost:6379
          AWS_ENDPOINT_URL: http://localhost:4566

  security-scan:
    runs-on: ubuntu-latest
    needs: code-quality
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
          
      - name: Run CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        with:
          languages: javascript
          
      - name: Scan Docker image
        run: |
          docker build -t photoshare-api .
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            -v $(pwd):/path \
            aquasec/trivy image photoshare-api

  build-and-push:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, security-scan]
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
        
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Deploy to ECS Staging
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }}-staging \
            --service ${{ env.ECS_SERVICE }}-staging \
            --force-new-deployment
            
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster ${{ env.ECS_CLUSTER }}-staging \
            --services ${{ env.ECS_SERVICE }}-staging

  e2e-tests:
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          BASE_URL: https://staging-api.photoshare.com
          
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: e2e-test-results
          path: e2e-results/

  performance-tests:
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Run load tests
        run: |
          docker run --rm \
            -v $(pwd)/tests/load:/tests \
            loadimpact/k6 run /tests/load-test.js
        env:
          BASE_URL: https://staging-api.photoshare.com

  deploy-production:
    runs-on: ubuntu-latest
    needs: [e2e-tests, performance-tests]
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Blue-Green Deployment
        run: |
          chmod +x ./scripts/blue-green-deploy.sh
          ./scripts/blue-green-deploy.sh
        env:
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image-tag }}
          
      - name: Health check
        run: |
          chmod +x ./scripts/health-check.sh
          ./scripts/health-check.sh https://api.photoshare.com
          
      - name: Rollback on failure
        if: failure()
        run: |
          chmod +x ./scripts/rollback.sh
          ./scripts/rollback.sh
```

#### Feature Branch Workflow
```yaml
name: Feature Branch CI
on:
  pull_request:
    branches: [main, develop]

jobs:
  validate-feature:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run feature tests
        run: npm run test:feature
        
      - name: Build preview
        run: npm run build
        
      - name: Deploy preview environment
        run: |
          chmod +x ./scripts/deploy-preview.sh
          ./scripts/deploy-preview.sh ${{ github.event.number }}
          
      - name: Comment PR with preview link
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🚀 Preview environment deployed: https://pr-${{ github.event.number }}.staging.photoshare.com`
            })
```

## Infrastructure as Code

### Terraform Configuration

#### Main Infrastructure
```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "photoshare-terraform-state"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "PhotoShare"
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = "DevOps Team"
    }
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "app_name" {
  description = "Application name"
  type        = string
  default     = "photoshare"
}

# Local values
locals {
  name_prefix = "${var.app_name}-${var.environment}"
  
  vpc_cidr = {
    dev     = "10.0.0.0/16"
    staging = "10.1.0.0/16" 
    prod    = "10.2.0.0/16"
  }
  
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

# VPC Module
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${local.name_prefix}-vpc"
  cidr = local.vpc_cidr[var.environment]
  
  azs             = local.azs
  private_subnets = [for k, v in local.azs : cidrsubnet(local.vpc_cidr[var.environment], 8, k)]
  public_subnets  = [for k, v in local.azs : cidrsubnet(local.vpc_cidr[var.environment], 8, k + 10)]
  database_subnets = [for k, v in local.azs : cidrsubnet(local.vpc_cidr[var.environment], 8, k + 20)]
  
  enable_nat_gateway   = true
  enable_vpn_gateway   = false
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

# ECS Module
module "ecs" {
  source = "./modules/ecs"
  
  name_prefix    = local.name_prefix
  vpc_id         = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnets
  public_subnet_ids  = module.vpc.public_subnets
  
  container_image = var.container_image
  container_port  = 3000
  
  min_capacity = var.environment == "prod" ? 2 : 1
  max_capacity = var.environment == "prod" ? 20 : 5
  
  environment_variables = {
    NODE_ENV = var.environment
    AWS_REGION = var.aws_region
  }
}

# RDS Module
module "rds" {
  source = "./modules/rds"
  
  name_prefix        = local.name_prefix
  vpc_id            = module.vpc.vpc_id
  subnet_ids        = module.vpc.database_subnets
  allowed_security_groups = [module.ecs.security_group_id]
  
  instance_class    = var.environment == "prod" ? "db.r6g.large" : "db.t3.micro"
  allocated_storage = var.environment == "prod" ? 100 : 20
  
  backup_retention_period = var.environment == "prod" ? 7 : 1
  multi_az               = var.environment == "prod"
  
  database_name = "photoshare"
  master_username = "photoshare_admin"
}

# ElastiCache Module
module "elasticache" {
  source = "./modules/elasticache"
  
  name_prefix        = local.name_prefix
  vpc_id            = module.vpc.vpc_id
  subnet_ids        = module.vpc.private_subnets
  allowed_security_groups = [module.ecs.security_group_id]
  
  node_type         = var.environment == "prod" ? "cache.r6g.large" : "cache.t3.micro"
  num_cache_nodes   = var.environment == "prod" ? 3 : 1
  
  automatic_failover_enabled = var.environment == "prod"
  multi_az_enabled          = var.environment == "prod"
}

# S3 Module
module "s3" {
  source = "./modules/s3"
  
  name_prefix = local.name_prefix
  environment = var.environment
  
  enable_versioning = var.environment == "prod"
  lifecycle_rules   = var.environment == "prod"
}

# Output values
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "ecs_cluster_name" {
  value = module.ecs.cluster_name
}

output "rds_endpoint" {
  value = module.rds.endpoint
  sensitive = true
}

output "elasticache_endpoint" {
  value = module.elasticache.endpoint
  sensitive = true
}

output "s3_bucket_name" {
  value = module.s3.bucket_name
}
```

#### ECS Module
```hcl
# terraform/modules/ecs/main.tf
resource "aws_ecs_cluster" "main" {
  name = "${var.name_prefix}-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = {
    Name = "${var.name_prefix}-cluster"
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

# Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.name_prefix}-app"
  network_mode             = "awsvpc"
  requires_compatibility   = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  task_role_arn           = aws_iam_role.ecs_task_role.arn
  
  container_definitions = jsonencode([
    {
      name  = "app"
      image = var.container_image
      
      portMappings = [
        {
          containerPort = var.container_port
          protocol      = "tcp"
        }
      ]
      
      environment = [
        for key, value in var.environment_variables : {
          name  = key
          value = value
        }
      ]
      
      secrets = [
        {
          name      = "DATABASE_URL"
          valueFrom = aws_secretsmanager_secret.database_url.arn
        },
        {
          name      = "REDIS_URL" 
          valueFrom = aws_secretsmanager_secret.redis_url.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.app.name
          awslogs-region        = data.aws_region.current.name
          awslogs-stream-prefix = "ecs"
        }
      }
      
      essential = true
      
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:${var.container_port}/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
  
  tags = {
    Name = "${var.name_prefix}-task-definition"
  }
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "${var.name_prefix}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.min_capacity
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight           = 100
  }
  
  network_configuration {
    security_groups  = [aws_security_group.ecs_tasks.id]
    subnets         = var.private_subnet_ids
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = var.container_port
  }
  
  depends_on = [aws_lb_listener.app]
  
  tags = {
    Name = "${var.name_prefix}-service"
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_cpu_policy" {
  name               = "${var.name_prefix}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

resource "aws_appautoscaling_policy" "ecs_memory_policy" {
  name               = "${var.name_prefix}-memory-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    target_value = 80.0
  }
}
```

## Deployment Strategies

### Blue-Green Deployment Script
```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -e

CLUSTER_NAME="${ECS_CLUSTER:-photoshare-cluster}"
SERVICE_NAME="${ECS_SERVICE:-photoshare-service}"
IMAGE_TAG="${IMAGE_TAG:-latest}"
REGION="${AWS_REGION:-us-east-1}"

echo "Starting blue-green deployment..."
echo "Cluster: $CLUSTER_NAME"
echo "Service: $SERVICE_NAME"
echo "Image Tag: $IMAGE_TAG"

# Get current task definition
CURRENT_TASK_DEF=$(aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --query 'services[0].taskDefinition' \
  --output text \
  --region $REGION)

echo "Current task definition: $CURRENT_TASK_DEF"

# Get current task definition JSON
aws ecs describe-task-definition \
  --task-definition $CURRENT_TASK_DEF \
  --query 'taskDefinition' \
  --region $REGION > current-task-def.json

# Update image in task definition
NEW_TASK_DEF=$(cat current-task-def.json | \
  jq --arg IMAGE "$IMAGE_TAG" '.containerDefinitions[0].image = $IMAGE' | \
  jq 'del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .placementConstraints, .compatibilities, .registeredAt, .registeredBy)')

# Register new task definition
NEW_TASK_DEF_ARN=$(echo $NEW_TASK_DEF | \
  aws ecs register-task-definition \
  --cli-input-json file:///dev/stdin \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text \
  --region $REGION)

echo "New task definition registered: $NEW_TASK_DEF_ARN"

# Update service with new task definition
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --task-definition $NEW_TASK_DEF_ARN \
  --region $REGION

echo "Service updated with new task definition"

# Wait for deployment to complete
echo "Waiting for deployment to stabilize..."
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --region $REGION

echo "Deployment completed successfully!"

# Cleanup
rm -f current-task-def.json
```

### Health Check Script
```bash
#!/bin/bash
# scripts/health-check.sh

URL="${1:-https://api.photoshare.com}"
MAX_RETRIES=30
RETRY_INTERVAL=10

echo "Performing health check on $URL/health"

for i in $(seq 1 $MAX_RETRIES); do
  echo "Attempt $i/$MAX_RETRIES..."
  
  if curl -f -s "$URL/health" > /dev/null; then
    echo "Health check passed!"
    
    # Additional checks
    echo "Checking API endpoints..."
    
    # Check authentication endpoint
    if curl -f -s "$URL/api/auth/status" > /dev/null; then
      echo "Authentication service: OK"
    else
      echo "Authentication service: FAIL"
      exit 1
    fi
    
    # Check database connectivity
    if curl -f -s "$URL/api/health/database" > /dev/null; then
      echo "Database connectivity: OK"
    else
      echo "Database connectivity: FAIL"
      exit 1
    fi
    
    # Check cache connectivity
    if curl -f -s "$URL/api/health/cache" > /dev/null; then
      echo "Cache connectivity: OK"
    else
      echo "Cache connectivity: FAIL"
      exit 1
    fi
    
    echo "All health checks passed!"
    exit 0
  else
    echo "Health check failed, retrying in $RETRY_INTERVAL seconds..."
    sleep $RETRY_INTERVAL
  fi
done

echo "Health check failed after $MAX_RETRIES attempts"
exit 1
```

### Rollback Script
```bash
#!/bin/bash
# scripts/rollback.sh

set -e

CLUSTER_NAME="${ECS_CLUSTER:-photoshare-cluster}"
SERVICE_NAME="${ECS_SERVICE:-photoshare-service}"
REGION="${AWS_REGION:-us-east-1}"

echo "Starting rollback procedure..."

# Get service history
DEPLOYMENTS=$(aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --query 'services[0].deployments' \
  --region $REGION)

# Find previous stable deployment
PREVIOUS_TASK_DEF=$(echo $DEPLOYMENTS | \
  jq -r '.[] | select(.status == "PRIMARY" and .rolloutState == "COMPLETED") | .taskDefinition' | \
  head -n 2 | tail -n 1)

if [ -z "$PREVIOUS_TASK_DEF" ]; then
  echo "No previous stable deployment found!"
  exit 1
fi

echo "Rolling back to: $PREVIOUS_TASK_DEF"

# Update service to previous task definition
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --task-definition $PREVIOUS_TASK_DEF \
  --region $REGION

echo "Rollback initiated, waiting for stabilization..."

# Wait for rollback to complete
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --region $REGION

echo "Rollback completed successfully!"

# Send notification
aws sns publish \
  --topic-arn "$SNS_TOPIC_ARN" \
  --message "PhotoShare: Rollback completed for $SERVICE_NAME in $CLUSTER_NAME" \
  --subject "PhotoShare Rollback Notification" \
  --region $REGION
```

## Container Strategy

### Multi-stage Dockerfile
```dockerfile
# Dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY src/ ./src/

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine AS production

# Add non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs package*.json ./

# Install security updates
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init curl && \
    rm -rf /var/cache/apk/*

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "dist/index.js"]
```

### Docker Compose for Development
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/photoshare_dev
      - REDIS_URL=redis://redis:6379
      - AWS_ENDPOINT_URL=http://localstack:4566
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - postgres
      - redis
      - localstack
    command: npm run dev
    
  postgres:
    image: postgres:14-alpine
    environment:
      - POSTGRES_DB=photoshare_dev
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
      
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,sns,lambda,dynamodb
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - localstack_data:/tmp/localstack
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  postgres_data:
  redis_data:
  localstack_data:
```

## Environment Management

### Environment Configuration
```javascript
// config/environment.js
const environments = {
  development: {
    api: {
      port: 3000,
      cors: {
        origin: ['http://localhost:3000', 'http://localhost:3001'],
        credentials: true
      }
    },
    database: {
      url: process.env.DATABASE_URL,
      pool: { min: 2, max: 10 },
      ssl: false
    },
    redis: {
      url: process.env.REDIS_URL,
      retryDelayOnFailover: 100
    },
    aws: {
      region: 'us-east-1',
      endpoint: process.env.AWS_ENDPOINT_URL, // For LocalStack
      s3: {
        bucket: 'photoshare-dev-photos'
      }
    },
    logging: {
      level: 'debug',
      format: 'combined'
    }
  },
  
  staging: {
    api: {
      port: 3000,
      cors: {
        origin: ['https://staging.photoshare.com'],
        credentials: true
      }
    },
    database: {
      url: process.env.DATABASE_URL,
      pool: { min: 5, max: 20 },
      ssl: { rejectUnauthorized: false }
    },
    redis: {
      url: process.env.REDIS_URL,
      tls: {},
      retryDelayOnFailover: 100
    },
    aws: {
      region: 'us-east-1',
      s3: {
        bucket: 'photoshare-staging-photos'
      }
    },
    logging: {
      level: 'info',
      format: 'json'
    }
  },
  
  production: {
    api: {
      port: 3000,
      cors: {
        origin: ['https://photoshare.com'],
        credentials: true
      }
    },
    database: {
      url: process.env.DATABASE_URL,
      pool: { min: 10, max: 50 },
      ssl: { rejectUnauthorized: true }
    },
    redis: {
      url: process.env.REDIS_URL,
      tls: {},
      retryDelayOnFailover: 100
    },
    aws: {
      region: 'us-east-1',
      s3: {
        bucket: 'photoshare-prod-photos'
      }
    },
    logging: {
      level: 'warn',
      format: 'json'
    }
  }
};

module.exports = environments[process.env.NODE_ENV || 'development'];
```

### Secrets Management
```bash
# scripts/setup-secrets.sh
#!/bin/bash

ENVIRONMENT="${1:-staging}"
REGION="${AWS_REGION:-us-east-1}"

echo "Setting up secrets for $ENVIRONMENT environment..."

# Database URL
aws secretsmanager create-secret \
  --name "photoshare/$ENVIRONMENT/database-url" \
  --description "Database connection URL for PhotoShare $ENVIRONMENT" \
  --secret-string "postgresql://username:password@hostname:5432/database" \
  --region $REGION

# JWT Secret
JWT_SECRET=$(openssl rand -base64 32)
aws secretsmanager create-secret \
  --name "photoshare/$ENVIRONMENT/jwt-secret" \
  --description "JWT signing secret for PhotoShare $ENVIRONMENT" \
  --secret-string "$JWT_SECRET" \
  --region $REGION

# Redis URL
aws secretsmanager create-secret \
  --name "photoshare/$ENVIRONMENT/redis-url" \
  --description "Redis connection URL for PhotoShare $ENVIRONMENT" \
  --secret-string "rediss://hostname:6380" \
  --region $REGION

# Encryption Key
ENCRYPTION_KEY=$(openssl rand -base64 32)
aws secretsmanager create-secret \
  --name "photoshare/$ENVIRONMENT/encryption-key" \
  --description "Data encryption key for PhotoShare $ENVIRONMENT" \
  --secret-string "$ENCRYPTION_KEY" \
  --region $REGION

echo "Secrets created successfully for $ENVIRONMENT environment"
```
# 🚀 DevOps Project 01 — Deploy Django Application on AWS ECS & ECR

<div align="center">

![AWS](https://img.shields.io/badge/AWS-ECS%20%7C%20ECR%20%7C%20ALB-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerization-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Django](https://img.shields.io/badge/Django-Python%20Web%20Framework-092E20?style=for-the-badge&logo=django&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

**Containerize a Django Application → Push to AWS ECR → Deploy on AWS ECS Fargate → Expose via AWS ALB**

*Implemented by [@hmurafique](https://github.com/hmurafique)*

</div>

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Prerequisites](#-prerequisites)
- [Project Structure](#-project-structure)
- [Step-by-Step Implementation](#-step-by-step-implementation)
  - [Phase 1 — Setup & Installation](#phase-1--setup--installation)
  - [Phase 2 — Django App Preparation](#phase-2--django-app-preparation)
  - [Phase 3 — Docker Image Build & Test](#phase-3--docker-image-build--test)
  - [Phase 4 — AWS ECR Setup & Push](#phase-4--aws-ecr-setup--push)
  - [Phase 5 — AWS ECS Setup](#phase-5--aws-ecs-setup)
  - [Phase 6 — Load Balancer Setup](#phase-6--load-balancer-setup)
  - [Phase 7 — Deploy & Verify](#phase-7--deploy--verify)
- [Common Issues & Fixes](#-common-issues--fixes)
- [Cleanup](#-cleanup-important)
- [Lessons Learned](#-lessons-learned)

---

## 📌 Project Overview

This project demonstrates how to **Containerize a Django Web Application** and deploy it on **AWS using ECS (Elastic Container Service) and ECR (Elastic Container Registry)** with a proper **ALB (Application Load Balancer)** for production-grade traffic management.

### What We Built
- Dockerized a Django Application with **Gunicorn** as WSGI server
- Pushed the Docker Image to **AWS ECR** (private container registry)
- Created an **ECS Fargate Cluster** (serverless — no EC2 management needed)
- Set up an **AWS ALB** on Port 80
- Configured **AWS CloudWatch Logs** for container monitoring
- Added a `/health/` endpoint for health checks

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Internet                         │
└─────────────────────────┬───────────────────────────────┘
                          │ Port 80
┌─────────────────────────▼───────────────────────────────┐
│          Application Load Balancer (ALB)                │
│              django-app-alb                             │
│         Port 80 → Forward to Target Group               │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│           Target Group (django-app-tg)                  │
│           Health Check: /health/ → Port 8000            │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│        ECS Fargate Cluster (django-app-cluster)         │
│  ┌─────────────────────────────────────────────────┐    │
│  │          ECS Task (django-app-task)             │    │
│  │   Django Container (Port 8000)                  │    │
│  │   Gunicorn WSGI Server — 3 Workers              │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
         │ Pull Image                    │ Send Logs
┌────────▼──────────┐          ┌─────────▼──────────┐
│   AWS ECR Repo    │          │  CloudWatch Logs   │
│ hello-world-      │          │  /ecs/django-app   │
│ django-app:latest │          └────────────────────┘
└───────────────────┘
```

---

## 🛠️ Tech Stack

| Technology | Purpose | Version |
|------------|---------|---------|
| Django | Python Web Framework | 4.2.x |
| Gunicorn | WSGI Production Server | 23.0.0 |
| Docker | Containerization | 29.x |
| AWS ECR | Container Image Registry | — |
| AWS ECS Fargate | Serverless Container Orchestration | — |
| AWS ALB | Application Load Balancer | — |
| AWS CloudWatch | Logs & Monitoring | — |
| AWS IAM | Roles & Permissions | — |
| Python | Backend Language | 3.9 |

---

## ✅ Prerequisites

### Accounts & Access
- [ ] AWS Account with sufficient permissions
- [ ] GitHub Account
- [ ] AWS CLI configured

### Tools Required
```bash
# Verify all tools are installed
docker --version        # Docker 20.x+
aws --version           # AWS CLI 2.x
git --version           # Git 2.x
python3 --version       # Python 3.9+
```

### Install on Ubuntu/EC2
```bash
# System update
apt update && apt upgrade -y

# Docker
apt install docker.io -y
systemctl start docker
systemctl enable docker

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Git & Python
apt install git python3-pip -y
```

### AWS IAM Permissions Needed
Your IAM user must have:
- `AmazonEC2ContainerRegistryFullAccess`
- `AmazonECS_FullAccess`
- `ElasticLoadBalancingFullAccess`
- `IAMFullAccess`
- `CloudWatchLogsFullAccess`

---

## 📁 Project Structure

```
DevOps-Project-001/
├── Dockerfile                      # Multi-stage Docker build
├── manage.py                       # Django management script
├── requirements.txt                # Python dependencies
├── db.sqlite3                      # SQLite database
├── ecs-trust-policy.json           # IAM trust policy (created during setup)
├── task-definition.json            # ECS task definition (created during setup)
└── hello_world_django_app/
    ├── __init__.py
    ├── settings.py                 # Django settings (STATIC_ROOT added)
    ├── urls.py                     # URL routing (/health/ endpoint added)
    ├── views.py
    ├── wsgi.py
    └── asgi.py
```

---

## 📖 Step-by-Step Implementation

---

### Phase 1 — Setup & Installation

#### Step 1.1 — AWS Configure

Get Access Keys: `AWS Console → IAM → Users → Security Credentials → Access Keys → Create`

```bash
aws configure
# AWS Access Key ID: YOUR_ACCESS_KEY
# AWS Secret Access Key: YOUR_SECRET_KEY
# Default region name: us-east-1
# Default output format: json
```

Verify connection:
```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

#### Step 1.2 — Clone Project

```bash
git clone https://github.com/hmurafique/DevOps-Project-001.git
cd DevOps-Project-001
ls -la
```

---


### Phase 2 — Docker Image Build & Test

#### Step 2.1 — Build Docker Image

```bash
docker build -t hello-world-django-app:version-1 .
```

> ⏳ First build takes 3-5 minutes

Successful build ends with:
```
Successfully built 8f258303d352
Successfully tagged hello-world-django-app:version-1
125 static files copied to '/usr/src/app/static'
```

#### Step 2.2 — Verify Image

```bash
docker images | grep hello-world-django-app
```

#### Step 2.3 — Test Locally

```bash
# Run container
docker run -d -p 8000:8000 --name django-test hello-world-django-app:version-1

# Check container status
docker ps

# Check logs — should show gunicorn started
docker logs django-test

# Test health endpoint
curl http://localhost:8000/health/
# Expected: {"status": "healthy"}

# Cleanup
docker stop django-test && docker rm django-test
```

---

### Phase 3 — AWS ECR Setup & Push

#### Step 3.1 — Set Environment Variables

> ⚠️ If you open a new terminal session, you must re-run these exports!

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=us-east-1
export ECR_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/hello-world-django-app

echo "Account: $AWS_ACCOUNT_ID"
echo "ECR URI: $ECR_URI"
```

#### Step 3.2 — Create ECR Repository

```bash
aws ecr create-repository \
    --repository-name hello-world-django-app \
    --image-scanning-configuration scanOnPush=true \
    --image-tag-mutability MUTABLE \
    --region $AWS_REGION
```

> `scanOnPush=true` automatically scans for security vulnerabilities

#### Step 3.3 — Login to ECR

```bash
aws ecr get-login-password --region $AWS_REGION | \
docker login --username AWS --password-stdin \
${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

Expected: `Login Succeeded`

#### Step 3.4 — Tag & Push Image

```bash
# Tag for ECR
docker tag hello-world-django-app:version-1 $ECR_URI:latest

# Push to ECR
docker push $ECR_URI:latest
```

#### Step 3.5 — Verify Image in ECR

```bash
aws ecr describe-images \
    --repository-name hello-world-django-app \
    --region $AWS_REGION
```

Expected: `"imageStatus": "ACTIVE"`

---

### Phase 4 — AWS ECS Setup

#### Step 4.1 — Create IAM Role for ECS

```bash
# Trust policy
cat > ecs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
    --role-name ecsTaskExecutionRole \
    --assume-role-policy-document file://ecs-trust-policy.json

# Attach ECS policy
aws iam attach-role-policy \
    --role-name ecsTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Attach CloudWatch policy (IMPORTANT — without this tasks fail!)
aws iam attach-role-policy \
    --role-name ecsTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# Verify
aws iam list-attached-role-policies --role-name ecsTaskExecutionRole \
    --query 'AttachedPolicies[*].PolicyName'
```

Expected:
```json
["AmazonECSTaskExecutionRolePolicy", "CloudWatchLogsFullAccess"]
```

#### Step 4.2 — Get Network Info

```bash
# Default VPC
export VPC_ID=$(aws ec2 describe-vpcs \
    --filters "Name=isDefault,Values=true" \
    --query 'Vpcs[0].VpcId' --output text --region $AWS_REGION)

# List all subnets (pick one)
aws ec2 describe-subnets \
    --filters "Name=defaultForAz,Values=true" \
    --query 'Subnets[*].SubnetId' --output text --region $AWS_REGION

# Default Security Group
export SG_ID=$(aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=default" \
    --query 'SecurityGroups[0].GroupId' --output text --region $AWS_REGION)

# Set your subnet ID (from output above)
export SUBNET_ID=subnet-xxxxxxxxxxxxxxxxx

echo "VPC: $VPC_ID | SG: $SG_ID | Subnet: $SUBNET_ID"
```

#### Step 4.3 — Create ECS Cluster

```bash
aws ecs create-cluster \
    --cluster-name django-app-cluster \
    --region $AWS_REGION
```

Expected: `"status": "ACTIVE"`

#### Step 4.4 — Create Task Definition

```bash
cat > task-definition.json << EOF
{
    "family": "django-app-task",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "django-app",
            "image": "${ECR_URI}:latest",
            "portMappings": [
                {
                    "containerPort": 8000,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/django-app",
                    "awslogs-region": "${AWS_REGION}",
                    "awslogs-stream-prefix": "ecs",
                    "awslogs-create-group": "true"
                }
            },
            "healthCheck": {
                "command": ["CMD-SHELL", "curl -f http://localhost:8000/health/ || exit 1"],
                "interval": 30,
                "timeout": 10,
                "retries": 3,
                "startPeriod": 40
            }
        }
    ]
}
EOF

# Register task definition
aws ecs register-task-definition \
    --cli-input-json file://task-definition.json \
    --region $AWS_REGION

# Verify
aws ecs describe-task-definition \
    --task-definition django-app-task --region $AWS_REGION \
    --query 'taskDefinition.{Family:family,Status:status,Revision:revision}'
```

Expected:
```json
{"Family": "django-app-task", "Status": "ACTIVE", "Revision": 1}
```

---

### Phase 5 — Load Balancer Setup

> ⚠️ **Important:** Load Balancer MUST be configured before creating the ECS Service. AWS does not allow adding ALB to an existing service.

#### Step 5.1 — Create ALB

```bash
# Replace SUBNET_1 SUBNET_2 SUBNET_3 with your actual subnet IDs
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name django-app-alb \
    --subnets SUBNET_1 SUBNET_2 SUBNET_3 \
    --security-groups $SG_ID \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4 \
    --region $AWS_REGION \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text)

echo "ALB ARN: $ALB_ARN"
```

#### Step 5.2 — Create Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
    --name django-app-tg \
    --protocol HTTP \
    --port 8000 \
    --vpc-id $VPC_ID \
    --target-type ip \
    --health-check-protocol HTTP \
    --health-check-path /health/ \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 10 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3 \
    --region $AWS_REGION \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

echo "Target Group ARN: $TG_ARN"
```

#### Step 5.3 — Create ALB Listener

```bash
LISTENER_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN \
    --region $AWS_REGION \
    --query 'Listeners[0].ListenerArn' \
    --output text)

echo "Listener ARN: $LISTENER_ARN"
```

#### Step 5.4 — Open Ports in Security Group

```bash
# Port 80 for ALB
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID --protocol tcp --port 80 \
    --cidr 0.0.0.0/0 --region $AWS_REGION

# Port 8000 for container
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID --protocol tcp --port 8000 \
    --cidr 0.0.0.0/0 --region $AWS_REGION
```

---

### Phase 6 — Deploy & Verify

#### Step 6.1 — Create ECS Service with ALB

```bash
aws ecs create-service \
    --cluster django-app-cluster \
    --service-name django-app-service \
    --task-definition django-app-task:1 \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=$TG_ARN,containerName=django-app,containerPort=8000" \
    --health-check-grace-period-seconds 60 \
    --region $AWS_REGION \
    --query 'service.{Name:serviceName,Status:status}'
```

#### Step 6.2 — Wait & Check Status

```bash
# Wait 3 minutes for task to start
sleep 180

# Check service
aws ecs describe-services \
    --cluster django-app-cluster \
    --services django-app-service \
    --region $AWS_REGION \
    --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}'
```

Expected:
```json
{"Status": "ACTIVE", "Running": 1, "Desired": 1}
```

If `Running: 0` — check events:
```bash
aws ecs describe-services \
    --cluster django-app-cluster \
    --services django-app-service \
    --region $AWS_REGION \
    --query 'services[0].events[0:3]'
```

#### Step 6.3 — Get ALB URL & Test

```bash
# Get ALB DNS
ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $ALB_ARN \
    --region $AWS_REGION \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

echo "=========================================="
echo "App URL   : http://$ALB_DNS"
echo "Health URL: http://$ALB_DNS/health/"
echo "=========================================="

# Test
curl http://$ALB_DNS/health/
```

Expected: `{"status": "healthy"}` ✅

---

## 🐛 Common Issues & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessDeniedException: logs:CreateLogGroup` | Missing CloudWatch permission on IAM role | `aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess` |
| `collectstatic` fails in Docker build | `STATIC_ROOT` not in settings.py | `echo "STATIC_ROOT = BASE_DIR / 'static'" >> settings.py` |
| `/health/` returns 404 | URL not configured | Add `health_check` view to urls.py |
| `myproject.wsgi` module not found | Wrong WSGI path in Dockerfile | Change to `hello_world_django_app.wsgi:application` |
| Service stuck at `Running: 0` | Task failing silently | Check events with `describe-services` |
| `EntityAlreadyExists` for IAM role | Role already exists | Safe to ignore — policies still attach |
| Variables empty after new terminal | Env vars reset on new session | Re-run all `export` commands |
| ALB cannot be added to service | AWS limitation | Delete service → create new service with ALB |

---

## 🛑 Cleanup (Important!)

> Run this after you're done to avoid AWS charges!

```bash
# 1. Stop ECS tasks
aws ecs update-service \
    --cluster django-app-cluster \
    --service django-app-service \
    --desired-count 0 --region $AWS_REGION

# 2. Delete ECS Service
aws ecs delete-service \
    --cluster django-app-cluster \
    --service django-app-service \
    --force --region $AWS_REGION

# 3. Delete ALB
aws elbv2 delete-load-balancer \
    --load-balancer-arn $ALB_ARN --region $AWS_REGION

# 4. Delete Target Group
sleep 10 && aws elbv2 delete-target-group \
    --target-group-arn $TG_ARN --region $AWS_REGION

# 5. Delete ECS Cluster
aws ecs delete-cluster \
    --cluster django-app-cluster --region $AWS_REGION

# 6. Delete ECR Repository
aws ecr delete-repository \
    --repository-name hello-world-django-app \
    --force --region $AWS_REGION

echo "✅ Cleanup complete!"
```

---

## 💡 Lessons Learned

1. **Multi-stage Docker builds** — Builder stage installs all dependencies, production stage only copies what's needed → smaller, secure image
2. **CloudWatch permission is critical** — Without `CloudWatchLogsFullAccess`, ECS tasks silently fail with `AccessDeniedException`
3. **ALB must be set at service creation** — Cannot add Load Balancer to existing ECS service; must delete and recreate
4. **Health endpoints are mandatory** — Without `/health/` returning 200, ALB marks targets as unhealthy and routes no traffic
5. **Environment variables reset** — Every new terminal session loses exported variables; always re-export
6. **Fargate is easier for beginners** — No EC2 instance management, AWS handles the infrastructure
7. **`awslogs-create-group: true`** — Allows ECS to auto-create CloudWatch log groups, no manual creation needed

---

## 📊 Skills Gained

```
Docker & Containerization    ████████████████████  100%
AWS ECR                      ████████████████████  100%
AWS ECS Fargate              ████████████████████  100%
Application Load Balancer    ████████████████████  100%
IAM Roles & Policies         ████████████████░░░░   80%
CloudWatch Logs              ████████████░░░░░░░░   60%
Django Production Setup      ████████████████░░░░   80%
```

---

## 🔗 References

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [Django Deployment Checklist](https://docs.djangoproject.com/en/4.2/howto/deployment/checklist/)
- [Gunicorn Documentation](https://gunicorn.org/)

---

## 🔜 What's Next

- [ ] Add HTTPS with ACM SSL Certificate
- [ ] Use RDS PostgreSQL instead of SQLite
- [ ] Add Auto Scaling Policy for ECS Service
- [ ] CI/CD with GitHub Actions → ECR → ECS
- [ ] Infrastructure as Code with Terraform

---

<div align="center">

**👨‍💻 Implemented by [@hmurafique](https://github.com/hmurafique Hafiz Muhammad Umar Rafique)**


⭐ **If this helped you, give it a star!**

</div>

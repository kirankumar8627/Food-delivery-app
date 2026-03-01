# 🍕 FoodRush — Food Delivery Application

A full-stack, cloud-native food delivery application built with React, Node.js, and MySQL. Deployed on AWS EKS using Terraform for infrastructure, Jenkins for CI/CD, and Prometheus + Grafana for monitoring.

---

## 🏗️ Architecture

``     
<img width="1408" height="768" alt="Gemini_Generated_Image_ngyq5ongyq5ongyq" src="https://github.com/user-attachments/assets/271a7583-0760-4f9a-8e34-4bd70e760d61" />

```

### Network Design

| Subnet Type | CIDR | Components |
|------------|------|-----------|
| Public  | 10.10.1.0/24, 10.10.2.0/24 | ALB, NAT Gateway, Internet Gateway |
| Private | 10.10.10.0/24, 10.10.11.0/24 | EKS Worker Nodes, RDS MySQL |

---

## 📁 Project Structure

```
fooddelivery/
├── app/
│   ├── backend/                    # Node.js + Express REST API
│   │   ├── server.js               # App entry point + Prometheus metrics
│   │   ├── config/
│   │   │   ├── db.js               # MySQL connection pool
│   │   │   └── init.sql            # DB schema + seed data
│   │   ├── routes/
│   │   │   ├── restaurants.js      # CRUD: restaurants
│   │   │   ├── menu.js             # CRUD: menu items
│   │   │   ├── orders.js           # Place orders, track status
│   │   │   └── users.js            # Register, login, JWT auth
│   │   ├── Dockerfile              # Multi-stage Node.js image
│   │   └── package.json
│   └── frontend/                   # React.js customer app
│       ├── src/App.js              # Full app: browse, order, track
│       ├── Dockerfile              # Multi-stage React + Nginx image
│       └── nginx.conf
├── terraform/
│   ├── main.tf                     # Root module
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
│       ├── vpc/                    # VPC, subnets, IGW, NAT GW
│       ├── sg/                     # Security Groups (ALB, EKS, RDS)
│       ├── eks/                    # EKS cluster + node group + IAM
│       └── rds/                    # RDS MySQL + subnet group
├── kubernetes/
│   └── base/
│       └── manifests.yaml          # Namespace, ConfigMap, Secret,
│                                   # Backend+Frontend Deployments,
│                                   # Services, HPA, ALB Ingress
├── jenkins/
│   └── Jenkinsfile                 # 10-stage CI/CD pipeline
├── ansible/
│   ├── playbooks/site.yml          # Master playbook
│   ├── inventory/hosts.ini         # Server inventory
│   └── roles/
│       ├── prometheus/             # Prometheus install + config + alert rules
│       ├── grafana/                # Grafana install + datasource provisioning
│       ├── alertmanager/           # Alertmanager + email/Slack routing
│       └── node-exporter/          # Node Exporter on all servers
└── README.md
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React.js, Nginx |
| Backend | Node.js, Express, JWT Auth |
| Database | MySQL 8.0 on AWS RDS |
| Cloud | AWS (EKS, RDS, ALB, VPC, IAM, ECR, Route53) |
| IaC | Terraform with S3 remote state + DynamoDB locking |
| Containers | Docker (multi-stage builds) |
| Orchestration | Kubernetes on EKS (HPA, Ingress, Secrets) |
| CI/CD | Jenkins Pipeline as Code |
| Code Quality | SonarQube with quality gates |
| Security Scan | Trivy (filesystem + image scanning) |
| Monitoring | Prometheus + Grafana + Alertmanager |
| Config Mgmt | Ansible (roles-based) |

---

## 🚀 Step-by-Step Deployment

### Prerequisites
```bash
aws --version      # AWS CLI v2
terraform --version # >= 1.5.0
kubectl version
docker --version
ansible --version  # >= 2.14
```

### 1. Configure AWS
```bash
aws configure
# AWS Access Key ID, Secret Key, Region: us-east-1
```

### 2. Create Terraform state backend
```bash
aws s3 mb s3://kiran-tf-state --region us-east-1
aws dynamodb create-table \
  --table-name tf-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST --region us-east-1
```

### 3. Provision AWS Infrastructure
```bash
cd terraform
terraform init
terraform plan -var="db_password=FoodRush@2024"
terraform apply -var="db_password=FoodRush@2024" -auto-approve

# Note outputs
terraform output eks_cluster_name
terraform output rds_endpoint
```

### 4. Build & Push Docker Images to ECR
```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
ECR="${ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com"

# Create ECR repos
aws ecr create-repository --repository-name fooddelivery-backend
aws ecr create-repository --repository-name fooddelivery-frontend

# Login
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR

# Build & Push
docker build -t $ECR/fooddelivery-backend:latest ./app/backend && docker push $ECR/fooddelivery-backend:latest
docker build -t $ECR/fooddelivery-frontend:latest ./app/frontend && docker push $ECR/fooddelivery-frontend:latest
```

### 5. Deploy to Kubernetes
```bash
# Configure kubectl
aws eks update-kubeconfig --name fooddelivery-dev-cluster --region us-east-1

# Update kubernetes/base/manifests.yaml:
# - Replace YOUR_ECR_REPO with your ECR registry URL
# - Replace YOUR_RDS_ENDPOINT with terraform output rds_endpoint
# - Replace YOUR_DB_PASSWORD with your actual password

kubectl apply -f kubernetes/base/manifests.yaml
kubectl rollout status deployment/backend -n fooddelivery
kubectl rollout status deployment/frontend -n fooddelivery

# Get app URL
kubectl get ingress fooddelivery-ingress -n fooddelivery
```

### 6. Initialize Database
```bash
# Port-forward to backend to run migrations
kubectl port-forward deployment/backend 5000:5000 -n fooddelivery &
# OR connect to RDS directly and run:
mysql -h YOUR_RDS_ENDPOINT -u admin -p fooddelivery < app/backend/config/init.sql
```

### 7. Deploy Monitoring with Ansible
```bash
# Update ansible/inventory/hosts.ini with your actual server IPs

# Test connectivity
ansible all -i ansible/inventory/hosts.ini -m ping

# Deploy full monitoring stack
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/site.yml

# Access dashboards
# Prometheus:   http://YOUR_PROMETHEUS_IP:9090
# Grafana:      http://YOUR_GRAFANA_IP:3000  (admin / FoodDelivery@2024)
# Alertmanager: http://YOUR_PROMETHEUS_IP:9093
```

---

## 🔄 CI/CD Pipeline Stages

```
1. Checkout          → Pull code from GitHub
2. Install & Test    → npm ci + unit tests (parallel: backend + frontend)
3. SonarQube         → Code quality analysis
4. Quality Gate      → Block if quality thresholds not met
5. Trivy FS Scan     → Filesystem vulnerability scan
6. Docker Build      → Build images (parallel: backend + frontend)
7. Trivy Image Scan  → Scan Docker images for CRITICAL CVEs
8. Push to ECR       → Push images to AWS ECR
9. Deploy to EKS     → Rolling update on Kubernetes
10. Smoke Test       → Health check after deployment
```

On **failure**: Auto-rollback via `kubectl rollout undo` + email alert
On **success**: Email notification to devops team

---

## 📊 Monitoring & Alerts

### Metrics Collected
- CPU, Memory, Disk usage per server (Node Exporter)
- HTTP request rate, latency, error rate (prom-client in backend)
- HTTP endpoint availability (Blackbox Exporter)

### Alert Rules
| Alert | Condition | Severity |
|-------|-----------|----------|
| InstanceDown | Server unreachable > 1 min | Critical |
| HighCPU | CPU > 85% for 5 min | Warning |
| HighMemory | Memory > 85% for 5 min | Warning |
| DiskSpaceLow | Disk > 80% | Warning |
| FoodDeliveryAPIDown | HTTP probe fails > 2 min | Critical |
| HighOrderErrorRate | 5xx > 5% of requests | Warning |
| SlowAPIResponse | p95 latency > 2s | Warning |

---

## 🧹 Cleanup
```bash
kubectl delete namespace fooddelivery
cd terraform && terraform destroy -var="db_password=FoodRush@2024"
```

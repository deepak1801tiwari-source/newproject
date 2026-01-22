Part 1: Setting Up Your Development Environment
Step 1.1: Install Required Tools
On Ubuntu/Debian:
bash# Update system
sudo apt update && sudo apt upgrade -y

# Install Git
sudo apt install git -y

# Install Node.js and npm
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y

# Verify installation
node --version  # Should show v18.x.x
npm --version   # Should show 9.x.x

# Install Python
sudo apt install python3 python3-pip -y

# Install Docker
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Terraform
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# IMPORTANT: Log out and log back in for Docker permissions to take effect
On macOS:
bash# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install all tools
brew install git node python@3.11 awscli terraform kubectl docker

# Start Docker Desktop (install from docker.com if not present)
Step 1.2: Verify All Installations
bash# Run these commands and make sure they all work:
git --version
node --version
npm --version
python3 --version
aws --version
terraform --version
kubectl version --client
docker --version

Part 2: AWS Account Setup
Step 2.1: Create AWS Account

Go to https://aws.amazon.com
Click "Create an AWS Account"
Follow the registration process
IMPORTANT: Add a payment method (required)
Complete phone verification

Step 2.2: Create IAM User (DON'T USE ROOT ACCOUNT!)
bash# Log into AWS Console as root user
# Go to: IAM → Users → Add User

# User details:
Name: cloudops-admin
Access type: ✓ Programmatic access

# Permissions:
Attach existing policies directly:
- AdministratorAccess (for learning; use least privilege in production)

# Download credentials CSV file - KEEP THIS SAFE!
Step 2.3: Configure AWS CLI
bash# Configure with your IAM user credentials
aws configure

# You'll be prompted for:
AWS Access Key ID: [paste from CSV]
AWS Secret Access Key: [paste from CSV]
Default region name: us-east-1
Default output format: json

# Test it works:
aws sts get-caller-identity
# Should show your account info
Step 2.4: Set Up Billing Alerts (CRITICAL!)
bash# Enable billing alerts in AWS Console:
# 1. Go to: Account → Billing Preferences
# 2. Check "Receive Billing Alerts"
# 3. Save preferences

# Create SNS topic for alerts
aws sns create-topic --name billing-alerts

# Subscribe your email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:billing-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com

# Confirm subscription from email

# Create billing alarm
aws cloudwatch put-metric-alarm \
  --alarm-name billing-alert-50 \
  --alarm-description "Alert when bill exceeds $50" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:billing-alerts

Part 3: Create Project Structure
Step 3.1: Create GitHub Repository
bash# Go to github.com and create new repository:
# Name: cloudops-platform
# Description: Enterprise cloud platform project
# Public/Private: Your choice
# ✓ Add README
# ✓ Add .gitignore (Node)
# Click "Create repository"
Step 3.2: Clone and Setup Local Project
bash# Clone your repo
git clone https://github.com/YOUR_USERNAME/cloudops-platform.git
cd cloudops-platform

# Create entire directory structure in one command:
mkdir -p \
  terraform/modules/{vpc,eks,rds,s3} \
  services/{user-service,product-service,cart-service,order-service}/src \
  k8s/{user-service,product-service,cart-service,order-service} \
  lambda/{order-processor,email-notifier,image-resizer,analytics-processor} \
  .github/workflows \
  monitoring \
  docs \
  scripts \
  tests/{unit,integration}

# Verify structure
tree -L 2  # or 'ls -R' if tree not installed
Step 3.3: Create .gitignore File
bash# Create .gitignore in project root
cat > .gitignore << 'EOF'
# Terraform
*.tfstate
*.tfstate.*
.terraform/
*.tfvars
terraform.tfvars

# Node
node_modules/
npm-debug.log
package-lock.json
.env
.env.local

# Python
__pycache__/
*.pyc
*.pyo
venv/
env/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Secrets
*.pem
*.key
secrets/
credentials/

# Build
dist/
build/
*.zip

# Logs
*.log
logs/
EOF

Part 4: Create Terraform Files
Step 4.1: Create Backend Configuration
bashcd terraform

# Create backend.tf
cat > backend.tf << 'EOF'
terraform {
  backend "s3" {
    bucket         = "cloudops-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
EOF
Step 4.2: Create Main Terraform File
bash# Create main.tf
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "CloudOps-Platform"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
EOF
Step 4.3: Create Variables File
bash# Create variables.tf
cat > variables.tf << 'EOF'
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "db_username" {
  description = "Database master username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}
EOF
Step 4.4: Create terraform.tfvars (NEVER COMMIT THIS!)
bash# Create terraform.tfvars with actual values
cat > terraform.tfvars << 'EOF'
aws_region   = "us-east-1"
environment  = "dev"
vpc_cidr     = "10.0.0.0/16"
db_username  = "dbadmin"
db_password  = "YourStrongPassword123!"  # CHANGE THIS!
EOF

# Make sure it's in .gitignore!
grep terraform.tfvars ../.gitignore

Part 5: Create Microservices
Step 5.1: Create User Service
bashcd ../services/user-service

# Initialize npm project
npm init -y

# Install dependencies
npm install express pg redis jsonwebtoken bcryptjs @aws-sdk/client-secrets-manager dotenv

# Install dev dependencies
npm install --save-dev nodemon jest eslint

# Create src/index.js
# Copy the user service code from the previous artifact I created
File: services/user-service/src/index.js
bash# Use your text editor or VS Code to create this file
# Copy the complete user service code I provided earlier
code src/index.js  # Opens VS Code
# OR
nano src/index.js  # Terminal editor
Step 5.2: Create Dockerfile for User Service
bash# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/

FROM node:18-alpine
WORKDIR /app
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
COPY --from=builder --chown=nodejs:nodejs /app .
USER nodejs
EXPOSE 3001
HEALTHCHECK --interval=30s --timeout=3s CMD node -e "require('http').get('http://localhost:3001/health',(r)=>{process.exit(r.statusCode===200?0:1)})"
CMD ["node", "src/index.js"]
EOF
Step 5.3: Create .dockerignore
bashcat > .dockerignore << 'EOF'
node_modules
npm-debug.log
.env
.git
.gitignore
README.md
Dockerfile
.dockerignore
EOF
Step 5.4: Update package.json Scripts
bash# Edit package.json to add scripts
nano package.json

# Add these to the "scripts" section:
{
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "test": "jest",
    "lint": "eslint src/"
  }
}
Step 5.5: Repeat for Other Services
bash# Product Service
cd ../product-service
npm init -y
npm install express pg @aws-sdk/client-s3 multer
# Create Dockerfile, src/index.js (copy from my previous artifact)

# Cart Service (similar structure)
cd ../cart-service
# ... same process

# Order Service
cd ../order-service
# ... same process

Part 6: Create Kubernetes Manifests
Step 6.1: Create ConfigMap
bashcd ../../k8s

cat > configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  redis-host: "redis-cluster.default.svc.cluster.local"
  aws-region: "us-east-1"
  s3-bucket: "dev-shopcloud-data"
  log-level: "info"
EOF
Step 6.2: Create Secrets Template
bashcat > secrets.yaml << 'EOF'
# DO NOT COMMIT WITH REAL VALUES!
apiVersion: v1
kind: Secret
metadata:
  name: database-secrets
  namespace: default
type: Opaque
stringData:
  host: "REPLACE_WITH_RDS_ENDPOINT"
  username: "dbadmin"
  password: "REPLACE_WITH_PASSWORD"
EOF
Step 6.3: Create User Service Deployment
bashcd user-service

# Copy the deployment.yaml content from my previous artifact
# Create deployment.yaml, service.yaml, hpa.yaml

Part 7: Create Lambda Functions
Step 7.1: Order Processor Lambda
bashcd ../../lambda/order-processor

# Create handler.py
# Copy the Python code from my previous artifact

# Create requirements.txt
cat > requirements.txt << 'EOF'
boto3==1.28.0
EOF
Step 7.2: Create Deployment Package
bash# Install dependencies locally
pip3 install -r requirements.txt -t .

# Create zip file
zip -r function.zip . -x "*.git*" "*.pyc" "__pycache__/*"

# This creates function.zip ready for upload to AWS

Part 8: Create CI/CD Pipeline
Step 8.1: Create GitHub Actions Workflow
bashcd ../../.github/workflows

# Create microservice-deploy.yml
# Copy the GitHub Actions workflow from my previous artifact
Step 8.2: Add GitHub Secrets
bash# Go to GitHub repository:
# Settings → Secrets and variables → Actions → New repository secret

# Add these secrets:
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_ACCOUNT_ID=your_12_digit_account_id
SLACK_WEBHOOK_URL=https://hooks.slack.com/... (optional)

Part 9: Testing Locally
Step 9.1: Create docker-compose.yml
bashcd ../../

# Create docker-compose.yml for local testing
# Copy the docker-compose configuration from earlier
Step 9.2: Test Services Locally
bash# Start all services
docker-compose up -d

# Check if running
docker-compose ps

# Test user service
curl http://localhost:3001/health

# View logs
docker-compose logs -f user-service

# Stop all
docker-compose down

Part 10: Deploy to AWS
Step 10.1: Create Terraform Backend
bashcd terraform

# Create S3 bucket for state
aws s3api create-bucket \
  --bucket cloudops-terraform-state \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket cloudops-terraform-state \
  --versioning-configuration Status=Enabled

# Create DynamoDB table
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
Step 10.2: Initialize and Deploy Infrastructure
bash# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# See what will be created
terraform plan

# Apply changes (type 'yes' when prompted)
terraform apply

# Save outputs
terraform output > ../docs/terraform-outputs.txt
Step 10.3: Configure kubectl for EKS
bash# Get EKS cluster name from Terraform output
EKS_CLUSTER=$(terraform output -raw eks_cluster_name)

# Update kubeconfig
aws eks update-kubeconfig --name $EKS_CLUSTER --region us-east-1

# Test connection
kubectl get nodes
kubectl get namespaces

Part 11: Push to GitHub
Step 11.1: Commit and Push
bashcd ..

# Add all files
git add .

# Commit
git commit -m "Initial project setup with complete infrastructure"

# Push to GitHub
git push origin main

# GitHub Actions will automatically trigger!
# Check: github.com/YOUR_USERNAME/cloudops-platform/actions

Part 12: Monitoring & Verification
Step 12.1: Check Deployments
bash# Check Kubernetes pods
kubectl get pods

# Check services
kubectl get svc

# Check logs
kubectl logs -l app=user-service --tail=50

# Check EKS cluster
aws eks describe-cluster --name dev-eks-cluster
Step 12.2: Access Services
bash# Get load balancer URL
kubectl get ingress

# Test API
INGRESS_URL=$(kubectl get ingress api-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$INGRESS_URL/api/users/health

🎉 Congratulations!
You now have a fully functional CloudOps Platform!
Next Steps:

✅ Complete documentation in /docs
✅ Create architecture diagram
✅ Set up monitoring dashboards
✅ Add more Lambda functions
✅ Implement ML pipeline
✅ Create demo video

Cleanup (When Done):
bash# Destroy all infrastructure to avoid charges
cd terraform
terraform destroy

# Delete S3 buckets manually (Terraform won't delete non-empty buckets)
aws s3 rb s3://cloudops-terraform-state --force

🆘 Common Issues & Solutions
Issue 1: "terraform: command not found"
bash# Solution: Add terraform to PATH
export PATH=$PATH:/usr/local/bin
# Or reinstall Terraform
Issue 2: "AWS credentials not found"
bash# Solution: Reconfigure AWS CLI
aws configure
Issue 3: "Permission denied" when running Docker
bash# Solution: Add user to docker group and restart
sudo usermod -aG docker $USER
# Log out and log back in
Issue 4: Pods stuck in "Pending" state
bash# Check what's wrong
kubectl describe pod <pod-name>
# Usually means not enough resources or image pull issues
Issue 5: High AWS costs
bash# Check current costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics UnblendedCost

# Stop expensive resources
terraform destroy

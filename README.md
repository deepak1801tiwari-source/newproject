Step 1: Install Required Software
For Ubuntu/Linux:
bash# Update your system
sudo apt update && sudo apt upgrade -y


# Install Git (for code versioning)
sudo apt install git -y


# Install Node.js (for running JavaScript)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y


# Install Python
sudo apt install python3 python3-pip -y


# Install Docker (for containers)
sudo apt install docker.io -y
sudo systemctl start docker
sudo usermod -aG docker $USER


# Install AWS CLI (to control AWS)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install


# Install Terraform (for infrastructure)
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# ⚠️ IMPORTANT: Log out and log back in for Docker to work!
For Mac:
bash# Install Homebrew (package manager)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install everything at once
brew install git node python@3.11 awscli terraform kubectl

# Install Docker Desktop from docker.com
Verify Everything Works:
bashgit --version
node --version
python3 --version
aws --version
terraform --version
docker --version
✅ If all commands show version numbers, you're ready!

PHASE 2: Setup AWS Account ☁️
Step 2: Create AWS Account

Go to https://aws.amazon.com
Click "Create an AWS Account"
Enter your email and create password
Add payment method (💳 required, but we'll set spending limits!)
Verify your phone number

Step 3: Create IAM User (IMPORTANT!)
⚠️ Never use your root account for daily work!
bash# 1. Log into AWS Console (https://console.aws.amazon.com)
# 2. Search for "IAM" in the search bar
# 3. Click "Users" → "Create User"
# 4. Username: cloudops-admin
# 5. Check: ✓ Provide user access to AWS Management Console
# 6. Click "Next"
# 7. Select "Attach policies directly"
# 8. Search and select: AdministratorAccess
# 9. Click "Create User"
# 10. Download the CSV file with credentials - SAVE IT SAFELY!
Step 4: Configure AWS on Your Computer
bash# Run this command
aws configure

# You'll be asked for:
# AWS Access Key ID: [paste from CSV file]
# AWS Secret Access Key: [paste from CSV file]
# Default region: us-east-1
# Output format: json

# Test it works
aws sts get-caller-identity
# You should see your account info!
Step 5: Setup Billing Alerts 🚨
This is CRITICAL to avoid surprise charges!
bash# 1. Go to AWS Console → Search "Billing"
# 2. Click "Billing Preferences"
# 3. Check ✓ "Receive Billing Alerts"
# 4. Save

# Create alert topic
aws sns create-topic --name billing-alerts

# Subscribe your email (REPLACE with your email!)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:billing-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com

# Check your email and confirm subscription!

# Create $50 spending alert
aws cloudwatch put-metric-alarm \
  --alarm-name billing-alert-50 \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold

PHASE 3: Create Your Project 📁
Step 6: Create GitHub Repository

Go to https://github.com
Click "New repository"
Name: cloudops-platform
Check ✓ Add README
Check ✓ Add .gitignore (select Node)
Click "Create repository"

Step 7: Clone and Setup Locally
bash# Clone your repo (replace YOUR_USERNAME!)
git clone https://github.com/YOUR_USERNAME/cloudops-platform.git
cd cloudops-platform

# Create all folders at once
mkdir -p \
  terraform/modules/{vpc,eks,rds,s3} \
  services/{user-service,product-service,cart-service,order-service}/src \
  k8s/{user-service,product-service,cart-service,order-service} \
  lambda/{order-processor,email-notifier} \
  .github/workflows \
  monitoring \
  docs \
  scripts

# See your structure
ls -R
Step 8: Create .gitignore File
bashcat > .gitignore << 'EOF'
# Secrets - NEVER commit these!
*.tfvars
.env
*.pem
*.key
secrets/

# Terraform
*.tfstate
.terraform/

# Node
node_modules/
package-lock.json

# Python
__pycache__/
venv/

# IDE
.vscode/
.idea/

# OS
.DS_Store
EOF

PHASE 4: Build Your First Service 🔧
Step 9: Create User Service
bashcd services/user-service

# Initialize Node.js project
npm init -y

# Install dependencies
npm install express pg redis jsonwebtoken bcryptjs
npm install --save-dev nodemon
Create src/index.js:
bashmkdir src
nano src/index.js  # or use VS Code: code src/index.js
Paste this simple code:
javascriptconst express = require('express');
const app = express();
const PORT = 3001;

app.use(express.json());

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', service: 'user-service' });
});

// Get all users (dummy data)
app.get('/api/users', (req, res) => {
  res.json({
    users: [
      { id: 1, name: 'John Doe', email: 'john@example.com' },
      { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
    ]
  });
});

// Create user
app.post('/api/users', (req, res) => {
  const { name, email } = req.body;
  res.status(201).json({
    id: 3,
    name,
    email,
    created: new Date()
  });
});

app.listen(PORT, () => {
  console.log(`🚀 User service running on port ${PORT}`);
});
Update package.json scripts:
bashnano package.json
Add this to the "scripts" section:
json"scripts": {
  "start": "node src/index.js",
  "dev": "nodemon src/index.js"
}
Test it locally:
bashnpm start
Open another terminal:
bashcurl http://localhost:3001/health
# Should return: {"status":"healthy","service":"user-service"}
🎉 Your first service is working!

PHASE 5: Containerize Your Service 🐳
Step 10: Create Dockerfile
bash# Stop your service (Ctrl+C)
# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy source code
COPY src/ ./src/

# Expose port
EXPOSE 3001

# Run the service
CMD ["node", "src/index.js"]
EOF
Step 11: Build and Test Docker Image
bash# Build image
docker build -t user-service .

# Run container
docker run -p 3001:3001 user-service

# Test (in another terminal)
curl http://localhost:3001/health

# Stop container (Ctrl+C)

PHASE 6: Deploy to AWS ☁️
Step 12: Create Terraform Configuration
bashcd ../../terraform

# Create main configuration
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
  region = "us-east-1"
}

# Create a simple S3 bucket for testing
resource "aws_s3_bucket" "test_bucket" {
  bucket = "cloudops-test-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}

output "bucket_name" {
  value = aws_s3_bucket.test_bucket.id
}
EOF
Step 13: Deploy Infrastructure
bash# Initialize Terraform
terraform init

# See what will be created
terraform plan

# Create the infrastructure
terraform apply
# Type 'yes' when asked

# You should see:
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
✅ You just created your first cloud resource!

PHASE 7: Setup CI/CD 🔄
Step 14: Create GitHub Actions Workflow
bashcd ../.github/workflows

cat > deploy.yml << 'EOF'
name: Deploy User Service

on:
  push:
    branches: [ main ]
    paths:
      - 'services/user-service/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Test service
        run: |
          cd services/user-service
          npm install
          npm test || echo "No tests yet"
      
      - name: Build Docker image
        run: |
          cd services/user-service
          docker build -t user-service .
      
      - name: Success
        run: echo "✅ Deployment complete!"
EOF

PHASE 8: Push to GitHub 📤
bashcd ../..

# Add all files
git add .

# Commit
git commit -m "Initial setup: User service with Docker and Terraform"

# Push to GitHub
git push origin main

# Check GitHub Actions
# Go to: github.com/YOUR_USERNAME/cloudops-platform/actions
# You should see your workflow running!

🎉 CONGRATULATIONS!
You've successfully:

✅ Set up your development environment
✅ Created an AWS account with billing alerts
✅ Built your first microservice
✅ Containerized it with Docker
✅ Deployed infrastructure with Terraform
✅ Set up automated deployment with GitHub Actions


📚 What's Next?
Easy Next Steps:

Add more services (product, cart, order) - copy the user-service pattern
Add a database - create RDS in Terraform
Deploy to Kubernetes - create EKS cluster
Add monitoring - set up CloudWatch dashboards

Choose your path:

Frontend Developer? → Build a React UI to call these APIs
Backend Developer? → Add authentication, database connections
DevOps Enthusiast? → Deploy to Kubernetes, add monitoring
Cloud Architect? → Design multi-region deployment


🆘 Common Issues & Solutions
Issue: "Command not found"
bash# Solution: Make sure you logged out and back in after installation
# Or add to PATH:
export PATH=$PATH:/usr/local/bin
Issue: "Docker permission denied"
bash# Solution:
sudo usermod -aG docker $USER
# Log out and log back in
Issue: "AWS credentials not found"
bash# Solution: Reconfigure
aws configure
Issue: High AWS costs
bash# Check costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics UnblendedCost

# Destroy everything
cd terraform
terraform destroy

💰 Cost Management
To minimize costs:

✅ Set billing alerts (we did this!)
✅ Use free tier resources
✅ Destroy resources when not using: terraform destroy
✅ Monitor daily in AWS Console

Estimated monthly costs:

Learning/Testing: $0-10 (mostly free tier)
Small production: $50-100
Medium production: $200-500

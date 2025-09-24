# 🚀 MERN Deployment to AWS EKS using Jenkins CI/CD
[![Terraform](https://img.shields.io/badge/Terraform-1.9+-purple?logo=terraform)]() [![Kubernetes](https://img.shields.io/badge/Kubernetes-1.33+-blue?logo=kubernetes)]() [![Jenkins](https://img.shields.io/badge/Jenkins-Pipeline-red?logo=jenkins)]() [![Docker](https://img.shields.io/badge/Docker-Build%20&%20Push-blue?logo=docker)]() [![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20ECR%20%7C%20S3-orange?logo=amazon-aws)]() [![GitHub](https://img.shields.io/badge/GitHub-Repo%20%26%20CI-black?logo=github)]()

## 📌 Project Description
This project demonstrates a **CI/CD pipeline** to deploy a **Python Flask application** into **AWS EKS** using **Terraform, Docker, Kubernetes, and Jenkins**.

The pipeline provisions infra, containerizes the Flask app, pushes to **ECR**, and deploys to **EKS** with health checks.


---
## 🛠️ Technologies Used
- ☁️ **AWS** → EKS, ECR, S3, IAM, VPC  
- <img src="https://www.vectorlogo.zone/logos/terraformio/terraformio-icon.svg" width="20"/> **Terraform** → Infrastructure provisioning  
- 🐳 **Docker** → Flask app containerization  
- ☸️ **Kubernetes (kubectl)** → App deployment & service exposure  
- ⚙️ **Jenkins** → CI/CD automation  
- 💻 **GitHub** → Source code repository 

---

### 📦 Jenkins  Prerequisite Plugin Installation

Before proceeding with the pipeline setup or deployment process, ensure the following plugins are installed in your CI/CD environment (e.g., Jenkins).

#### ✅ Required Plugins

| Plugin Name        | Purpose |
|--------------------|---------|
| **Pipeline: AWS Steps**      | Allows you to use aws credentials to connect to deploy project. |

## ⚙️ Jenkins Credentials Setup
1️⃣ **AWS Credentials**  
- **ID**: `aws-credentials`  
- **Type**: AWS Credentials  
- **Access Key ID** / **Secret Access Key**  

3️⃣ **GitHub Credentials**  
- **ID**: `Github`  
- **Type**: Username with Password  
- **Username**: `praysap`  
- **Password / Token**: GitHub PAT  


---
## 🛠 Terraform Infrastructure Provisioning
Terraform is used to provision AWS infrastructure for the application, including VPC, ECR, and EKS.

### Project Directory Structure
```plaintext
coding-assignment-prt/
├── terraform/
│   ├── ecr.tf
│   ├── eks.tf
│   ├── outputs.tf
│   ├── vpc.tf
│   ├── backend.tf
│   ├── providers.tf
│   ├── user-data.sh
```
### Define AWS Provider
File: `terraform/providers.tf`
```
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
}
```

### Define Variables
File: `terraform/variables.tf`
```
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-west-2"
}

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
  default     = "sagar-eks-cluster"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
}

variable "azs" {
  description = "Availability Zones"
  type        = list(string)
  default     = ["ap-south-1a", "ap-south-1b"]
}

variable "ecr_repo_names" {
  description = "List of ECR repository names to create"
  type        = list(string)
  default     = ["sagar-app-repo"]
}
```

###  VPC
File: `terraform/vpc.tf`
```
# Create VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.cluster_name}-vpc"
  }
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = element(["ap-south-1a", "ap-south-1b"], count.index)  # Use the corrected AZs
  map_public_ip_on_launch = true
  tags = {
    Name = "public-subnet-${element(["a", "b"], count.index)}"
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.cluster_name}-igw"
  }
}

# Create Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-rt"
  }
}

# Associate Public Subnets with Public Route Table
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

###  ECR
File: `terraform/ecr.tf`
```
# Create ECR Repositories
resource "aws_ecr_repository" "repo" {
  for_each             = toset(var.ecr_repo_names)
  name                 = each.key
  image_tag_mutability = "MUTABLE" # Use "IMMUTABLE" for production

  image_scanning_configuration {
    scan_on_push = true # Enable automatic vulnerability scanning on push :cite[4]:cite[6]
  }

  encryption_configuration {
    encryption_type = "AES256" # Server-side encryption
  }

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}

# Optional: Lifecycle Policy to clean up old images
resource "aws_ecr_lifecycle_policy" "repo_policy" {
  for_each   = toset(var.ecr_repo_names)
  repository = aws_ecr_repository.repo[each.key].name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 30 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 30
      }
      action = {
        type = "expire"
      }
    }]
  })
}
```

<img width="1919" height="948" alt="image" src="https://github.com/user-attachments/assets/e96f79ab-08bd-458f-a76a-8afdaadffd19" />


###  EKS
File: `terraform/eks.tf`
```
# IAM Role for EKS Cluster
resource "aws_iam_role" "eks_cluster_role" {
  name = "${var.cluster_name}-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

# Attach AmazonEKSClusterPolicy to Cluster Role
resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

# Create EKS Cluster
resource "aws_eks_cluster" "cluster" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster_role.arn
  version  = "1.30" # Specify your desired Kubernetes version

  vpc_config {
    subnet_ids = aws_subnet.public[*].id # Place control plane in private subnets
    # endpoint_private_access = true # Uncomment for private cluster
    # endpoint_public_access  = false # Uncomment for private cluster
  }

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]
}

# IAM Role for EKS Node Group
resource "aws_iam_role" "eks_node_role" {
  name = "${var.cluster_name}-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

# Attach necessary policies to Node Role
resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_node_role.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_node_role.name
}

resource "aws_iam_role_policy_attachment" "ecr_read_only_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_node_role.name
}

# Create EKS Managed Node Group
resource "aws_eks_node_group" "nodes" {
  cluster_name    = aws_eks_cluster.cluster.name
  node_group_name = "managed-nodes"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = aws_subnet.public[*].id

  scaling_config {
    desired_size = 2
    max_size     = 3
    min_size     = 1
  }

  instance_types = ["t3.medium"]

  # Ensure the IAM Role permissions are created before and deleted after the Node Group handling
  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only_policy,
  ]
}
```
<img width="1914" height="633" alt="EKS-cluter-name" src="https://github.com/user-attachments/assets/8198947b-f0d9-48d5-b8f5-db942c798e59" />


###  EKS
File: `terraform/outputs.tf`
```
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "eks_cluster_name" {
  description = "Name of the EKS cluster"
  value       = aws_eks_cluster.cluster.name
}

output "eks_cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = aws_eks_cluster.cluster.endpoint
}

output "ecr_repository_urls" {
  description = "URLs of the ECR repositories"
  value       = { for k, v in aws_ecr_repository.repo : k => v.repository_url }
}

# Command to update kubeconfig for the created cluster
output "configure_kubectl" {
  description = "Configure kubectl to use the new EKS cluster"
  value       = "aws eks update-kubeconfig --region ${var.aws_region} --name ${aws_eks_cluster.cluster.name}"
}
```
## 🚀 Deployment Steps

**Configure AWS Credentials:** Ensure AWS credentials are set in ~/.aws/credentials or as environment variables.

**Initialize Terraform**
```
terraform init
```
![terraform init](images/terraform-init.png)

**Format and Validate:**
```
terraform fmt
terraform validate
```
![terraform fmt](images/terraform-fmt.png)
![terraform validate](images/terraform-validate.png)

**Preview Changes:**
```
terraform plan
```
![terraform plan](images/terraform-plan.png)

**Apply Configuration:**
```
terraform apply
```
![terraform apply](images/terraform-apply.png)

**Destroy Infrastructure** (when needed):
You can use destroy
```
terraform destroy
```
![terraform destroy](images/terraform-destroy.png)

---
### ☸️ Steps to Deploy an Application on Kubernetes


📌 **Apply Deployment**
```bash
kubectl apply -f k8s/deployment.yaml
```
🔍 **Verify Pods**
```bash
kubectl get pods
```
```
🔍 **Verify Service**
```bash
kubectl get svc
```
---
### 🚀 Jenkins Pipeline Configuration for `Coding assignment Prt`
1. **Log in to Jenkins**
2. **Click on “New Item”**
   - This is usually located on the left-hand side of the Jenkins dashboard
3. **Enter a name for your job**
   - Example: `Coding assignment Prt`
4. **Select “Pipeline” as the project type**
5. **Click “OK”**
   - This will take you to the configuration page for the new pipeline job

#### 📁 Pipeline Definition

- **Definition**: Pipeline script from SCM
- **SCM**: Git
- **Repository URL**: `https://github.com/praysap/Coding-Assignment-EKS.git`
- **Credentials**: `praysap/******`
- **Branch Specifier**: `main`
- **Script Path**: `Jenkinsfile`

#### ⚡ Trigger

- [x] GitHub hook trigger for GITScm polling 

#### 📝 Notes

- This configuration uses a declarative pipeline stored in the `main` branch under the file `Jenkinsfile`.
- Ensure that the **GitHub webhook** is properly configured in your GitHub repository settings to trigger Jenkins jobs automatically.

![Jenkins Configuration](./images/flask-cicd-configration.png)
---

### 📄 Jenkinsfile

```groovy
pipeline {
  agent any
  options {
    timestamps()
  }

  environment {
    AWS_REGION        = 'us-west-2'
    CLUSTER_NAME      = 'three-tier-cluster'
    NAMESPACE         = 'three-tier'
    ECR_REPO_FRONTEND = 'mern-frontend'
    ECR_REPO_BACKEND  = 'mern-backend'
    ECR_REGISTRY      = '863541429677.dkr.ecr.us-west-2.amazonaws.com' // Replace with your AWS Account ID
  }

  stages {

    stage('Init AWS + Vars') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            echo "✅ AWS credentials and region set"
          }
        }
      }
    }

    stage('Checkout Code') {
      steps {
        git branch: 'main', url: 'https://github.com/praysap/Coding-Assignment-EKS'
      }
    }

    stage('Configure kubeconfig') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              set -e
              aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}
            """
          }
        }
      }
    }

    stage('Ensure ECR Repositories') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              set -e
              aws ecr describe-repositories --repository-names ${ECR_REPO_FRONTEND} >/dev/null 2>&1 || \
                aws ecr create-repository --repository-name ${ECR_REPO_FRONTEND}

              aws ecr describe-repositories --repository-names ${ECR_REPO_BACKEND} >/dev/null 2>&1 || \
                aws ecr create-repository --repository-name ${ECR_REPO_BACKEND}
            """
          }
        }
      }
    }

    stage('Login to ECR') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${ECR_REGISTRY}
            """
          }
        }
      }
    }

    stage('Build Frontend Image') {
      steps {
        sh """
          cd frontend
          docker build -t ${ECR_REGISTRY}/${ECR_REPO_FRONTEND}:latest .
        """
      }
    }

    stage('Build Backend Image') {
      steps {
        sh """
          cd backend
          docker build -t ${ECR_REGISTRY}/${ECR_REPO_BACKEND}:latest .
        """
      }
    }

    stage('Push Images to ECR') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              docker push ${ECR_REGISTRY}/${ECR_REPO_FRONTEND}:latest
              docker push ${ECR_REGISTRY}/${ECR_REPO_BACKEND}:latest
            """
          }
        }
      }
    }

    stage('Apply Manifests to EKS') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              echo "🚀 Applying backend deployment and service..."
              kubectl apply -f k8s/backend.yml -n ${NAMESPACE}

              echo "⏳ Waiting for backend pods to be ready..."
              kubectl rollout status deployment/backend-deployment -n ${NAMESPACE} --timeout=180s

              echo "🚀 Applying frontend deployment and service..."
              kubectl apply -f k8s/frontend.yml -n ${NAMESPACE}

              echo "⏳ Waiting for frontend pods to be ready..."
              kubectl rollout status deployment/frontend-deployment -n ${NAMESPACE} --timeout=180s
            """
          }
        }
      }
    }
  }
}

```

### 🚀 Pipeline Overview

<img width="1919" height="967" alt="Pipeline-overview" src="https://github.com/user-attachments/assets/fcb10f40-e59a-40f5-adaf-aa62bf22b02e" />
<img width="1917" height="986" alt="Pipeline-sucess" src="https://github.com/user-attachments/assets/dcb1cdb1-bbb0-4291-86f0-761fc30f53de" />



##### EKS deployment
<img width="1668" height="111" alt="image" src="https://github.com/user-attachments/assets/e6eb103e-f971-4032-a2fd-052610b4618a" />


##### EKS pod
<img width="1586" height="113" alt="image" src="https://github.com/user-attachments/assets/2b041c07-e6cd-4060-b61e-7ed550cb8595" />


##### EKS service
<img width="1609" height="117" alt="image" src="https://github.com/user-attachments/assets/9415e2ec-1d27-44fa-bac1-a0988c96730a" />


##### EKS Cluster
<img width="1919" height="965" alt="Cluster-information" src="https://github.com/user-attachments/assets/9f25e4c8-6e0f-4aa8-af2b-a56c5c4b92ee" />


### **🌍 Production App Live:**
<img width="1919" height="711" alt="backend-working" src="https://github.com/user-attachments/assets/4910de85-953d-45cb-b389-858d0b8130e0" />

<img width="1914" height="993" alt="frontend-working" src="https://github.com/user-attachments/assets/5b6f42fe-5171-4687-aa14-002926ce005c" />




---
## 📜 Project Information

### 📄 License Details
This project is released under the MIT License, granting you the freedom to:
- 🔓 Use in commercial projects
- 🔄 Modify and redistribute
- 📚 Use as educational material

---


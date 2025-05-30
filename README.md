Thanks for sharing the challenge details. Based on the images and the hackathon structure, here's a breakdown of what you need to deliver and a full working setup template (assuming you're choosing the EKS Deployment Track — let me know if you're choosing Fargate or Lambda Container instead).


---

✅ Overview of Required Deliverables

1. Dockerize 2 Node.js microservices (Patient Service and Appointment Service).


2. Provision AWS infrastructure using Terraform:

VPC, subnets (across 2 AZs)

EKS Cluster & node groups

IAM roles/policies

S3 (for backend)

RDS or DynamoDB if required



3. Terraform Workflows using GitHub Actions:

terraform fmt, plan, and apply automation



4. CI/CD Pipeline for the app (GitHub Actions) – Docker build → Push to ECR → Deploy to EKS


5. Monitoring/Logging with CloudWatch


6. Documentation + Terraform state mgmt (S3 + DynamoDB)




---

🛠️ Code Components

I’ll now begin generating the full folder and code structure. It will include:

devops-hackathon/
├── microservices/
│   ├── patient-service/
│   │   ├── Dockerfile
│   │   └── app.js (placeholder)
│   └── appointment-service/
│       ├── Dockerfile
│       └── app.js (placeholder)
├── terraform/
│   ├── backend.tf
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
├── .github/
│   └── workflows/
│       ├── terraform.yml
│       └── cicd.yml
├── manifests/
│   ├── patient-deployment.yaml
│   ├── appointment-deployment.yaml
│   └── service.yaml
└── README.md

Let’s begin step by step:


---

1. Dockerfile (Same for both microservices)

microservices/patient-service/Dockerfile (same for appointment-service):

FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["node", "app.js"]


---

2. Sample app.js

Just to simulate functionality:

microservices/patient-service/app.js

const express = require('express');
const app = express();

app.get('/patient', (req, res) => {
  res.json({ message: 'Patient service is running' });
});

app.listen(3000, () => {
  console.log('Patient service listening on port 3000');
});


---

3. Terraform Setup for EKS

terraform/backend.tf

terraform {
  backend "s3" {
    bucket         = "your-terraform-backend-bucket"
    key            = "eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}

terraform/main.tf (summary version, can be expanded)

provider "aws" {
  region = var.region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"

  name = "hackathon-vpc"
  cidr = "10.0.0.0/16"
  azs             = ["${var.region}a", "${var.region}b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  enable_dns_hostnames = true
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "hackathon-cluster"
  cluster_version = "1.27"
  subnets         = module.vpc.public_subnets
  vpc_id          = module.vpc.vpc_id

  node_groups = {
    eks_nodes = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1

      instance_types = ["t3.medium"]
    }
  }
}

terraform/variables.tf

variable "region" {
  default = "us-east-1"
}


---

4. GitHub Actions Terraform Workflow

.github/workflows/terraform.yml

name: Terraform

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  terraform:
    name: Terraform CI
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.0

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init
        working-directory: terraform

      - name: Terraform Plan
        run: terraform plan
        working-directory: terraform

      - name: Terraform Apply (only on main)
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
        working-directory: terraform


---

5. CI/CD Pipeline Workflow

.github/workflows/cicd.yml

name: CI/CD Pipeline

on:
  push:
    paths:
      - 'microservices/**'
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push Docker image (Patient)
        run: |
          docker build -t patient-service ./microservices/patient-service
          docker tag patient-service:latest ${{ secrets.ECR_REPO }}/patient-service:latest
          docker push ${{ secrets.ECR_REPO }}/patient-service:latest

      # Add deploy to EKS here using kubectl or Helm


---

6. Kubernetes Manifest (simplified)

manifests/patient-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: patient-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: patient-service
  template:
    metadata:
      labels:
        app: patient-service
    spec:
      containers:
        - name: patient-service
          image: <ECR-IMAGE-URL>
          ports:
            - containerPort: 3000


---

7. Monitoring and Logging

Enable CloudWatch Container Insights on EKS cluster.

Use aws-for-fluent-bit as DaemonSet or sidecar for advanced logging.

Example ConfigMap and DaemonSet setup can be shared if required.



---

Would you like me to export this into a downloadable ZIP or push to a GitHub repo structure for you?

Also, please confirm:

Which deployment track are you choosing (EKS / Fargate / Lambda)?

Do you want help with writing README.md documentation too?

Should I extract and use code from the uploaded ZIP for microservices?


Let me know so I can tailor the code 100% for you.

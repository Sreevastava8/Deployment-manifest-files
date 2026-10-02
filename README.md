# ShopSphere Application, Kubernetes & CI/CD

## Project Overview

ShopSphere is a cloud-native e-commerce application built from a React/Vite frontend and Java Spring Boot microservices.

The application is containerized with Docker and deployed to **Amazon EKS** using Kubernetes manifests. Jenkins automates testing, Docker image creation, Amazon ECR publishing, environment selection, Kubernetes deployment, and rollout verification.

The application repository is designed to work with the AWS infrastructure repository.

---

# Application Architecture

```text
                         GitHub
                            |
                            v
                         Jenkins
                            |
                 +----------+----------+
                 |                     |
              DEV/QA                  PROD
                 |                     |
              Auto Deploy          Approval
                 |                     |
                 +----------+----------+
                            |
                            v
                          ECR
                            |
                            v
                         Amazon EKS
                            |
                    +-------+-------+
                    | Kubernetes   |
                    | Namespace    |
                    | `shopsphere` |
                    +-------+-------+
                            |
        +-------------------+-------------------+
        |          |          |          |      |
    Frontend    User      Product      Order  Payment
        |       Service     Service    Service Service
        +-------------------+-------------------+
                            |
                            v
                     AWS Load Balancer
                         Controller
                            |
                            v
                       AWS ALB
                            |
                         Internet
```

---

# Application Components

| Component | Technology | Purpose |
|---|---|---|
| Frontend | React / Vite / Nginx | Customer-facing web application |
| User Service | Java / Spring Boot / Maven | Authentication and user functionality |
| Product Service | Java / Spring Boot / Maven | Product/catalog functionality |
| Order Service | Java / Spring Boot / Maven | Order processing |
| Payment Service | Java / Spring Boot / Maven | Payment processing integration |

Ports used by the application services are defined in their Kubernetes manifests and application configuration.

---

# Repository Structure

```text
shopsphere-app-scr-k8s-files/
│
├── Jenkinsfile
│
├── application-scr/
│   ├── docker-compose.yml
│   │
│   ├── frontend/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │
│   ├── services/
│   │   ├── user-service/
│   │   ├── product-service/
│   │   ├── order-service/
│   │   └── payment-service/
│   │
│   ├── contracts/
│   ├── docs/
│   └── scripts/
│
└── kubernetes/
    ├── namespace/
    ├── config/
    ├── service-account/
    ├── ingress/
    ├── frontend/
    ├── user-service/
    ├── product-service/
    ├── order-service/
    └── payment-service/
```

---

# Dockerization

Each backend microservice has its own Docker image.

The frontend is also containerized.

Jenkins builds five application images:

```text
product-service
user-service
order-service
payment-service
frontend
```

Images are tagged using the Git commit:

```text
product-service-<commit>
user-service-<commit>
order-service-<commit>
payment-service-<commit>
frontend-<commit>
```

This gives each deployment a traceable image version.

---

# Amazon ECR

The application uses one ECR repository per environment:

```text
dev-shopsphere
qa-shopsphere
prod-shopsphere
```

The Jenkins pipeline dynamically determines the repository from the Git branch.

```text
develop → dev-shopsphere
qa      → qa-shopsphere
main    → prod-shopsphere
```

Images are pushed to:

```text
<account>.dkr.ecr.ap-south-1.amazonaws.com/<repository>
```

---

# Kubernetes Namespace

All ShopSphere application workloads run in:

```text
shopsphere
```

The pipeline creates/applies the namespace before deploying application resources.

---

# Kubernetes Resources

The deployment includes:

## Namespace

Creates the application namespace:

```text
shopsphere
```

## ConfigMap

Stores non-sensitive application configuration.

## Service Account

The application uses a dedicated Kubernetes service account:

```text
shopsphere-app
```

The service account is associated with an AWS IAM role.

## Deployments

Separate deployments are used for:

```text
frontend
user-service
product-service
order-service
payment-service
```

## Services

Kubernetes Services provide stable networking for the application workloads.

## Ingress

The Kubernetes Ingress provides external application routing through the AWS Load Balancer Controller.

---

# AWS Load Balancer Controller

The application does not require a manually created Kubernetes/NGINX load balancer.

The AWS Load Balancer Controller watches Kubernetes Ingress resources and creates an AWS Application Load Balancer.

```text
Client
  |
  v
AWS Application Load Balancer
  |
  v
Kubernetes Ingress
  |
  +--> frontend service
  +--> product service
  +--> user service
  +--> order service
  +--> payment service
```

---

# Secrets Management

Sensitive application values are not stored directly inside Kubernetes manifests or Git.

The deployment pipeline retrieves secret ARNs from AWS.

The main sources are:

```text
AWS Secrets Manager
       |
       +--> JWT secret
       |
       +--> RDS master user secret
```

The application uses:

```text
Secrets Store CSI Driver
        +
AWS Secrets Manager Provider
        +
EKS Workload IAM
```

The Kubernetes service account receives only the permissions required to access the application's secrets.

The actual secret values are not printed by the Jenkins pipeline.

---

# Environment Strategy

The same application pipeline supports three environments.

```text
develop
   |
   v
DEV
```

```text
qa
   |
   v
QA
```

```text
main
   |
   v
PROD
```

Jenkins determines the environment from `BRANCH_NAME`.

---

# Environment-Specific Resources

## DEV

```text
EKS:
shopsphere-dev-eks

ECR:
dev-shopsphere

IAM Role:
dev-shopsphere-pod

RDS:
shopsphere-dev-postgres

JWT Secret:
dev/shopsphere/jwt-secret
```

## QA

```text
EKS:
shopsphere-qa-eks

ECR:
qa-shopsphere

IAM Role:
qa-shopsphere-pod

RDS:
shopsphere-qa-postgres

JWT Secret:
qa/shopsphere/jwt-secret
```

## PROD

```text
EKS:
shopsphere-prod-eks

ECR:
prod-shopsphere

IAM Role:
prod-shopsphere-pod

RDS:
shopsphere-prod-postgres

JWT Secret:
prod/shopsphere/jwt-secret
```

These names are generated dynamically by the Jenkins pipeline rather than maintaining separate hardcoded pipelines for each environment.

---

# Jenkins Application Pipeline

The pipeline performs the following steps:

```text
Checkout
   |
Determine Environment
   |
Verify Tools
   |
AWS Authentication
   |
Backend Tests
   |
Frontend Build/Test
   |
Login to ECR
   |
Build Docker Images
   |
Push Images to ECR
   |
Configure EKS Access
   |
Install/Verify AWS Load Balancer Controller
   |
Install/Verify Secrets Store CSI
   |
Deploy Namespace
   |
Deploy ConfigMap
   |
Deploy Service Account
   |
Resolve Secret ARNs
   |
Resolve ALB Security Group
   |
Production Approval
   |
Deploy Applications
   |
Deploy Ingress
   |
Verify Rollouts
   |
Deployment Summary
```

---

# Jenkins Environment Selection

The central environment mapping is:

```groovy
if (env.BRANCH_NAME == 'develop') {
    env.DEPLOY_ENV = 'dev'
}
else if (env.BRANCH_NAME == 'qa') {
    env.DEPLOY_ENV = 'qa'
}
else if (env.BRANCH_NAME == 'main') {
    env.DEPLOY_ENV = 'prod'
}
```

From `DEPLOY_ENV`, Jenkins generates:

```groovy
env.EKS_CLUSTER =
    "shopsphere-${env.DEPLOY_ENV}-eks"

env.ECR_REPOSITORY =
    "${env.DEPLOY_ENV}-shopsphere"

env.POD_IAM_ROLE_NAME =
    "${env.DEPLOY_ENV}-shopsphere-pod"

env.RDS_IDENTIFIER =
    "shopsphere-${env.DEPLOY_ENV}-postgres"

env.JWT_SECRET_ID =
    "${env.DEPLOY_ENV}/shopsphere/jwt-secret"
```

This is what makes one Jenkinsfile reusable across all three environments.

---

# Jenkins Build and Test Flow

Before deployment, Jenkins verifies the required tools:

```text
Java
Maven
Node
NPM
Docker
AWS CLI
kubectl
Helm
```

Backend services are tested with Maven.

Example:

```bash
mvn -B clean test
```

The frontend is installed and built using:

```bash
npm ci
npm run build
```

A failed test/build stops the pipeline before deployment.

---

# Docker Build

Jenkins builds each service with a Git-based image tag.

Example:

```bash
docker build \
  -t "$ECR_REPOSITORY_URL:product-service-$IMAGE_TAG" \
  application-scr/services/product-service
```

The same approach is used for all services and the frontend.

---

# ECR Push

After successful image builds:

```bash
docker push "$ECR_REPOSITORY_URL:product-service-$IMAGE_TAG"
```

The pipeline pushes all five application images.

The Git commit becomes part of the image tag, making deployments traceable.

---

# EKS Deployment

Jenkins configures Kubernetes access dynamically:

```bash
aws eks update-kubeconfig \
    --region "$AWS_REGION" \
    --name "$EKS_CLUSTER"
```

The cluster is selected according to the Git branch.

Example:

```text
develop → shopsphere-dev-eks
qa      → shopsphere-qa-eks
main    → shopsphere-prod-eks
```

---

# Kubernetes Deployment Order

The pipeline applies the resources in a controlled order:

```text
1. Namespace
2. ConfigMap
3. Service Account
4. SecretProviderClass
5. Application Deployments
6. Application Services
7. Ingress
8. Rollout verification
```

---

# Production Approval

DEV and QA can deploy automatically after successful validation.

Production requires manual approval.

```text
main
  |
  v
Build
  |
Test
  |
Docker Build
  |
ECR Push
  |
Manual Approval
  |
Deploy PROD
```

This prevents a successful build from automatically changing production without an approval step.

---

# Deployment Verification

After deployment, Jenkins waits for Kubernetes rollouts.

Example:

```bash
kubectl -n "$NAMESPACE" rollout status \
    deployment/frontend \
    --timeout=5m
```

The pipeline verifies:

```text
frontend
product-service
user-service
order-service
payment-service
```

It also displays:

```bash
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get deployments
```

This provides a deployment summary in Jenkins.

---

# Failure Handling

The Jenkins pipeline has success and failure post actions.

On failure, the pipeline reports the environment and provides useful Kubernetes troubleshooting commands:

```bash
kubectl -n shopsphere get pods

kubectl -n shopsphere get events \
    --sort-by=.lastTimestamp

kubectl -n shopsphere describe pods
```

---

# Local Development

The application also supports local Docker Compose development.

Example:

```bash
cd application-scr

docker-compose up --build
```

This allows developers to test the application locally before the Jenkins/EKS deployment process.

---

# End-to-End CI/CD Flow

```text
Developer
    |
    v
Git Push / Merge
    |
    v
GitHub
    |
    v
Jenkins
    |
    +----------------------+
    |                      |
    v                      v
DEV / QA                 PROD
    |                      |
    |                  Approval
    |                      |
    +----------+-----------+
               |
               v
         Run Tests
               |
               v
         Docker Build
               |
               v
              ECR
               |
               v
          Private EKS
               |
               v
        Kubernetes Pods
               |
               v
        AWS ALB / Ingress
               |
               v
            Users
```

---

# Key DevOps Practices Demonstrated

This repository demonstrates:

- Git-based CI/CD
- Jenkins Declarative Pipeline
- Multi-environment deployment
- Docker containerization
- Amazon ECR
- Amazon EKS
- Kubernetes Deployments
- Kubernetes Services
- Kubernetes Ingress
- AWS Load Balancer Controller
- Secrets Store CSI Driver
- AWS Secrets Manager
- EKS workload IAM
- Environment-specific configuration
- Immutable Git commit-based image tags
- Automated rollout verification
- Production approval gates
- Failure diagnostics

---

# Repository Security

Do not commit:

```text
real passwords
JWT secret values
AWS access keys
private credentials
.env files containing secrets
Terraform state
Terraform plans
```

The pipeline should access AWS using a controlled Jenkins IAM identity rather than credentials stored in source code.

---

# Relationship With Infrastructure Repository

This application repository depends on the AWS infrastructure repository.

The infrastructure repository provides:

```text
VPC
EKS
ECR
RDS
IAM
Secrets Manager
ALB security groups
SSM access
```

The application repository consumes those resources to:

```text
Build application
       |
       v
Push images to ECR
       |
       v
Deploy to EKS
       |
       v
Expose through ALB
       |
       v
Connect to managed secrets/database
```

---

# Recruiter Summary

**ShopSphere is a containerized microservices application deployed on Amazon EKS through a Jenkins CI/CD pipeline. The pipeline runs application tests, builds Docker images, pushes immutable Git commit-based images to environment-specific ECR repositories, dynamically selects DEV/QA/PROD from Git branches, configures private EKS access, integrates AWS Load Balancer Controller and Secrets Store CSI Driver, retrieves secret references from AWS Secrets Manager, deploys Kubernetes workloads, verifies rollouts, and requires manual approval for production deployment.**

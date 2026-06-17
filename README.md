# Project 5 - GitOps Microservices Deployment on Amazon EKS using ArgoCD

## Project Status

✅ Terraform Infrastructure Provisioned

✅ Amazon EKS Cluster Created

✅ Managed Node Group Configured

✅ OIDC Provider Configured

✅ IAM Roles for Service Accounts (IRSA) Configured

✅ AWS Load Balancer Controller Installed

✅ ArgoCD Installed

✅ GitOps Workflow Implemented

✅ Kubernetes Deployments Created

✅ Kubernetes Services Created

✅ Kubernetes Ingress Configured

✅ Application Load Balancer Created

✅ Microservices Successfully Deployed

✅ End-to-End Application Accessible Through ALB

## Overview

This project demonstrates a complete GitOps workflow using Amazon EKS, ArgoCD, Kubernetes, AWS Load Balancer Controller, and GitHub.

The application consists of three microservices:

- User Service
- Product Service
- Order Service

ArgoCD continuously monitors this repository and automatically synchronizes any changes to the EKS cluster.

---

# Architecture

```text
GitHub Repository
        │
        ▼
     ArgoCD
        │
        ▼
   Amazon EKS
        │
 ┌──────┼──────┐
 ▼      ▼      ▼
User  Product Order
Svc    Svc     Svc
        │
        ▼
     Ingress
        │
        ▼
 AWS Application Load Balancer
```

---

# Repository Structure

```text
.
├── README.md
├── kustomization.yaml
├── namespace.yaml
├── ingress
│   └── ingress.yaml
├── order-service
│   ├── deployment.yaml
│   └── service.yaml
├── product-service
│   ├── deployment.yaml
│   └── service.yaml
└── user-service
    ├── deployment.yaml
    └── service.yaml
```

---

# Technologies Used

- Amazon EKS
- Kubernetes
- ArgoCD
- AWS Load Balancer Controller
- Application Load Balancer (ALB)
- Docker
- GitHub
- GitOps
- Kustomize

---

# Resources Created

## Namespace

File:

```text
namespace.yaml
```

Purpose:

Creates a dedicated namespace called `gitops-demo` to isolate application resources.

Resource Created:

```text
Namespace
```

---

## Deployments

Files:

```text
user-service/deployment.yaml
product-service/deployment.yaml
order-service/deployment.yaml
```

Purpose:

Creates Pods and manages replicas automatically.

Resources Created:

```text
Deployment
ReplicaSet
Pods
```

---

## Services

Files:

```text
user-service/service.yaml
product-service/service.yaml
order-service/service.yaml
```

Purpose:

Provides stable networking and service discovery for Pods.

Resources Created:

```text
ClusterIP Services
```

---

## Ingress

File:

```text
ingress/ingress.yaml
```

Purpose:

Routes external traffic from AWS ALB to Kubernetes services.

Resources Created:

```text
Ingress
Application Load Balancer
Target Groups
Listeners
Listener Rules
```

---

# ArgoCD Setup

## Create Namespace

Command:

```bash
kubectl create namespace argocd
```

What it does:

Creates a dedicated namespace for ArgoCD components.

---

## Install ArgoCD

Command:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

What it does:

Installs ArgoCD server, controller, repo-server, Redis, CRDs, RBAC resources, ConfigMaps, and Secrets.

Resources Created:

```text
Deployments
Services
ConfigMaps
Secrets
CRDs
Roles
RoleBindings
ClusterRoles
ClusterRoleBindings
```

---

## Verify Installation

Command:

```bash
kubectl get pods -n argocd
```

What it does:

Displays all ArgoCD pods and their status.

Expected Output:

```text
Running
```

---

## Retrieve Initial Admin Password

Command:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

What it does:

Retrieves and decodes the initial ArgoCD admin password.

---

---

# IAM OIDC Provider and IRSA Configuration

AWS Load Balancer Controller requires AWS API permissions to create:

- Application Load Balancers
- Target Groups
- Listeners
- Security Group Rules

Instead of storing AWS credentials inside Pods, IAM Roles for Service Accounts (IRSA) was used.

---

## Step 1 - Verify OIDC Issuer

Command:

```bash
aws eks describe-cluster \
  --name project5-eks-cluster \
  --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

What it does:

Retrieves the OIDC issuer URL associated with the EKS cluster.

Example:

```text
https://oidc.eks.ap-south-1.amazonaws.com/id/DDF6FA5668C55E1BFF5E718A8154F5BE
```

---

## Step 2 - Check Existing OIDC Providers

Command:

```bash
aws iam list-open-id-connect-providers
```

What it does:

Lists all IAM OIDC providers configured in the AWS account.

Purpose:

Used to verify whether the cluster OIDC provider already exists.

---

## Step 3 - Associate OIDC Provider

Command:

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster project5-eks-cluster \
  --approve
```

What it does:

Creates and associates an IAM OIDC provider with the EKS cluster.

Purpose:

Allows Kubernetes Service Accounts to assume IAM Roles using Web Identity.

Result:

```text
OIDC Provider Created
```

Verification:

```bash
aws iam list-open-id-connect-providers
```

---

# AWS Load Balancer Controller IAM Policy

## Download IAM Policy

Command:

```bash
curl -o iam_policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

What it does:

Downloads the AWS Load Balancer Controller IAM policy document.

---

## Create IAM Policy

Command:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

What it does:

Creates a custom IAM policy containing permissions required by the AWS Load Balancer Controller.

Resources Managed:

```text
ALB
Target Groups
Listeners
Security Groups
Tags
```

Verification:

```bash
aws iam list-policies --scope Local
```

---

# IAM Role for Service Account (IRSA)

## Create IAM Service Account

Command:

```bash
eksctl create iamserviceaccount \
  --cluster=project5-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name=Project5EKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region ap-south-1
```

What it does:

Creates:

```text
IAM Role
Trust Relationship
Kubernetes Service Account
```

Purpose:

Allows the AWS Load Balancer Controller Pod to securely access AWS APIs.

Resources Created:

```text
IAM Role
IAM Trust Policy
Kubernetes ServiceAccount
```

Verification:

```bash
kubectl get serviceaccount aws-load-balancer-controller -n kube-system
```

---

# AWS Load Balancer Controller Installation

## Add Helm Repository

Command:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

What it does:

Adds the official AWS EKS Helm repository.

---

## Update Helm Repository

Command:

```bash
helm repo update
```

What it does:

Downloads the latest chart information.

---

## Install AWS Load Balancer Controller

Command:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=project5-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<VPC_ID>
```

What it does:

Installs the AWS Load Balancer Controller into the EKS cluster.

Purpose:

Automatically provisions AWS Application Load Balancers when Kubernetes Ingress resources are created.

Resources Created:

```text
Deployment
Pods
ClusterRoles
ClusterRoleBindings
Webhook Configuration
```

Verification:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected:

```text
READY 2/2
AVAILABLE 2
```

---

# IRSA Workflow

```text
AWS Load Balancer Controller Pod
                │
                ▼
 Kubernetes Service Account
                │
                ▼
 IAM Role (IRSA)
                │
                ▼
 AssumeRoleWithWebIdentity
                │
                ▼
 AWS APIs
                │
                ▼
 Create ALB
 Create Target Groups
 Create Listeners
 Configure Routing
```

Benefits:

- No AWS Access Keys stored in Pods
- Least Privilege Access
- AWS Recommended Approach
- Secure Authentication

# ALB Creation through Kubernetes Ingress

## Create Ingress Resource

File:

```text
ingress/ingress.yaml
```

Purpose:

Routes external traffic from AWS ALB to Kubernetes services.

Example Flow:

```text
Internet
    │
    ▼
Application Load Balancer
    │
    ▼
Ingress
    │
 ┌──┼──┐
 ▼  ▼  ▼
User Product Order
Svc  Svc   Svc
```

Verification:

```bash
kubectl get ingress -n gitops-demo
```

Expected:

```text
ADDRESS
xxxxxxxx.ap-south-1.elb.amazonaws.com
```

What Happens Automatically?

```text
Ingress Created
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
Application Load Balancer
        │
        ▼
Target Groups
        │
        ▼
Listeners
        │
        ▼
Routing Rules
```

# GitOps Workflow

## Create ArgoCD Application

Purpose:

Connects ArgoCD to the GitHub repository.

Configuration:

```text
Repository URL
Path
Cluster URL
Namespace
```

Enabled:

```text
Auto Sync
Self Heal
Prune
```

What it does:

Whenever code is pushed to GitHub, ArgoCD automatically synchronizes changes to the EKS cluster.

---

# Kustomize

File:

```text
kustomization.yaml
```

Purpose:

Combines multiple Kubernetes manifests into a single deployment package.

Example:

```yaml
resources:
  - namespace.yaml
  - user-service/deployment.yaml
  - user-service/service.yaml
  - product-service/deployment.yaml
  - product-service/service.yaml
  - order-service/deployment.yaml
  - order-service/service.yaml
  - ingress/ingress.yaml
```

---

# Useful Commands

## View Nodes

```bash
kubectl get nodes
```

Purpose:

Displays worker nodes connected to the cluster.

---

## View Pods

```bash
kubectl get pods -A
```

Purpose:

Displays Pods across all namespaces.

---

## View Deployments

```bash
kubectl get deployments -A
```

Purpose:

Displays Deployments and replica status.

---

## View Services

```bash
kubectl get svc -A
```

Purpose:

Displays all Kubernetes Services.

---

## View Ingress

```bash
kubectl get ingress -n gitops-demo
```

Purpose:

Displays ALB DNS endpoint created by AWS Load Balancer Controller.

---

## Describe Pod

```bash
kubectl describe pod <pod-name> -n gitops-demo
```

Purpose:

Provides detailed troubleshooting information.

Includes:

```text
Events
Image Pull Errors
Container Status
Scheduling Information
Network Details
```

---

# Troubleshooting Performed

## Issue 1 - Unable to Access Cluster

Error:

```text
You must be logged in to the server
```

Root Cause:

No EKS access entry configured.

Fix:

Created EKS Access Entry and associated AmazonEKSAdminPolicy.

Result:

```text
kubectl access restored
```

---

## Issue 2 - Forbidden Access

Error:

```text
cannot list resource "nodes"
```

Root Cause:

IAM principal lacked Kubernetes RBAC permissions.

Fix:

Associated:

```text
AmazonEKSAdminPolicy
```

Result:

```text
kubectl commands worked successfully
```

---

## Issue 3 - OIDC Provider Missing

Problem:

AWS Load Balancer Controller installation failed.

Root Cause:

OIDC provider was not associated with the EKS cluster.

Fix:

```bash
eksctl utils associate-iam-oidc-provider \
--region ap-south-1 \
--cluster project5-eks-cluster \
--approve
```

Result:

OIDC provider created successfully.

---

## Issue 4 - IAM Role Conflict

Error:

```text
AmazonEKSLoadBalancerControllerRole already exists
```

Root Cause:

Role name conflict from previous cluster.

Fix:

Created a new role:

```text
Project5EKSLoadBalancerControllerRole
```

Result:

AWS Load Balancer Controller installed successfully.

---

## Issue 5 - Invalid Kustomization

Error:

```text
invalid Kustomization
```

Root Cause:

Used:

```yaml
*
```

instead of:

```yaml
-
```

inside kustomization.yaml.

Fix:

```yaml
resources:
  - namespace.yaml
```

Result:

ArgoCD synchronized successfully.

---

## Issue 6 - ImagePullBackOff

Error:

```text
ImagePullBackOff
```

Root Cause:

Incorrect image names specified in Deployment manifests.

Example:

```text
dhusinth/user-service:v1
```

Repository did not exist.

Fix:

Updated Deployment manifests with correct image names.

Result:

Pods entered Running state.

---

# Validation

## Verify Pods

```bash
kubectl get pods -n gitops-demo
```

Expected:

```text
Running
```

---

## Verify Services

```bash
kubectl get svc -n gitops-demo
```

Expected:

```text
ClusterIP services created
```

---

## Verify Ingress

```bash
kubectl get ingress -n gitops-demo
```

Expected:

```text
ALB DNS Name
```

---

## Verify Application

Open:

```text
http://<ALB-DNS>
```

Expected:

```text
User Service
Product Service
Order Service
```

Application accessible through AWS Application Load Balancer.

---

# Learning Outcomes

Through this project I gained hands-on experience with:

- Amazon EKS
- Kubernetes Deployments
- Kubernetes Services
- Kubernetes Ingress
- ArgoCD
- GitOps
- Kustomize
- AWS Load Balancer Controller
- IAM OIDC Provider
- Application Load Balancer
- Troubleshooting Kubernetes Deployments
- Production-style Microservices Deployment

---

# Project Achievements

This project successfully demonstrated:

- Infrastructure as Code using Terraform
- Kubernetes on Amazon EKS
- GitOps using ArgoCD
- Secure AWS Authentication using IRSA
- Automated ALB Provisioning
- Microservices Deployment
- Kubernetes Networking
- Ingress Based Routing
- Cloud Native Architecture

Final Deployment Flow:

```text
GitHub
   │
   ▼
ArgoCD
   │
   ▼
Amazon EKS
   │
   ▼
Deployments
   │
   ▼
Pods
   │
   ▼
Services
   │
   ▼
Ingress
   │
   ▼
AWS Application Load Balancer
   │
   ▼
End Users
```

# Author

Dhusinth

Project 5 - GitOps Microservices Deployment on Amazon EKS using ArgoCD
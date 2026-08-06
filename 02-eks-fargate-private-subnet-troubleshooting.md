# Amazon EKS: Managed Node Groups, Fargate, Private Subnet & NAT Gateway

## Overview

This project demonstrates deploying workloads on both Amazon EKS Managed Node Groups (EC2) and AWS Fargate while using a private subnet for Fargate workloads.

The main objective was to configure AWS Fargate pods in a private subnet and troubleshoot connectivity issues preventing container images from being pulled from public registries.

---

## Environment

- AWS EKS
- Kubernetes v1.36
- Region: us-east-2
- Managed Node Group (EC2)
- AWS Fargate
- Private Subnet: 172.31.128.0/24
- NAT Gateway
- Route Tables
- Security Groups
- Network ACLs

---

## Architecture

```text
                    Internet
                        |
                 Internet Gateway
                        |
                Public Route Table
                        |
                  NAT Gateway
                        |
                Private Route Table
                        |
          Private Subnet (172.31.128.0/24)
                        |
                AWS Fargate Pods

-------------------------------------------------

          EKS Managed Node Group (EC2)
                        |
                 Public Subnet(s)
                        |
                    nginx Pod
```

## Namespace Creation

```bash
kubectl create namespace demo-ns
```

---

## Deploying a Pod on EC2 Managed Nodes

```bash
kubectl run nginx \
  --image=nginx:latest \
  -n demo-ns
```

Verification:

```bash
kubectl get pods -n demo-ns -o wide
```

Result:

```text
nginx   1/1 Running
Node: ip-172-31-20-57
```

The pod was successfully scheduled on an EC2 worker node.

---

## Creating Fargate Profile

Namespace Selector:

```text
demo-ns
```

Label Selector:

```text
type=fargate
```

Private Subnet:

```text
172.31.128.0/24
```

---

## Deploying a Pod on AWS Fargate

```bash
kubectl run httpd \
  --image=httpd:latest \
  -n demo-ns \
  --labels type=fargate
```

Verification:

```bash
kubectl get pods -n demo-ns -o wide
```

Result:

```text
httpd
Node: fargate-ip-172-31-128-x
```

The pod was correctly scheduled on AWS Fargate.

---

## Issue Encountered

The pod failed with:

```text
ImagePullBackOff
ErrImagePull
```

Events showed:

```text
Failed to pull image
dial tcp xx.xx.xx.xx:443: i/o timeout
```

Although the pod was successfully scheduled on Fargate, it could not pull images from:

- Docker Hub
- Amazon ECR Public

---

## Troubleshooting Performed

### Verified Fargate Profile

```bash
aws eks describe-fargate-profile \
  --cluster-name demo-eks-cluster \
  --fargate-profile-name demo-eks-fargate
```

Status:

```text
ACTIVE
```

### Verified Route Tables

Private Route Table:

```text
172.31.0.0/16 -> local
0.0.0.0/0     -> NAT Gateway
```

Public Route Table:

```text
172.31.0.0/16 -> local
0.0.0.0/0     -> Internet Gateway
```

### Verified Security Groups

Outbound Rules:

```text
All Traffic
Destination: 0.0.0.0/0
```

### Verified Network ACLs

Inbound:

```text
Allow All Traffic
```

Outbound:

```text
Allow All Traffic
```

### Verified DNS Settings

```bash
aws ec2 describe-vpc-attribute \
  --vpc-id vpc-085ac02c3d54d0e8c \
  --attribute enableDnsSupport
```

```bash
aws ec2 describe-vpc-attribute \
  --vpc-id vpc-085ac02c3d54d0e8c \
  --attribute enableDnsHostnames
```

Result:

```text
EnableDnsSupport: true
EnableDnsHostnames: true
```

### Verified Fargate Execution Role

```bash
aws iam list-attached-role-policies \
  --role-name demo-eks-fargate-role
```

Confirmed:

```text
AmazonEKSFargatePodExecutionRolePolicy
```

attached successfully.

---

## Root Cause

The issue was caused by the NAT Gateway configuration.

Although:

- Route tables were configured correctly
- Security Groups were configured correctly
- Network ACLs were configured correctly
- DNS settings were correct

AWS Fargate workloads could not establish outbound HTTPS connectivity, preventing image downloads from public registries.

This resulted in:

```text
ImagePullBackOff
ErrImagePull
dial tcp xx.xx.xx.xx:443: i/o timeout
```

---

## Resolution

The NAT Gateway configuration was corrected/recreated.

After updating the NAT Gateway and validating the routing path:

```text
Private Subnet
    |
Private Route Table
    |
NAT Gateway
    |
Internet Gateway
    |
Internet
```

The Fargate pod successfully downloaded images and started.

Verification:

```bash
kubectl get pod -n demo-ns
```

Output:

```text
NAME    READY   STATUS

nginx   1/1     Running
httpd   1/1     Running
```

---

## Key Learnings

- AWS Fargate requires outbound internet connectivity to pull container images.
- ImagePullBackOff combined with `dial tcp ...:443: i/o timeout` commonly indicates a networking problem.
- Troubleshooting should include:
  - Fargate Profiles
  - Route Tables
  - NAT Gateway
  - Security Groups
  - Network ACLs
  - IAM Roles
  - DNS Configuration
- EC2-based workloads and Fargate workloads use different networking paths and can behave differently during troubleshooting.
- Understanding traffic flow from Pod → Subnet → Route Table → NAT Gateway → Internet Gateway is critical when operating EKS environments.

---

## Skills Demonstrated

- Amazon EKS
- Kubernetes
- AWS Fargate
- VPC Networking
- Route Tables
- NAT Gateway Configuration
- Security Groups
- Network ACLs
- IAM
- AWS CLI
- Kubernetes Troubleshooting
- Root Cause Analysis
- DevOps Operations
- Cloud Infrastructure Debugging

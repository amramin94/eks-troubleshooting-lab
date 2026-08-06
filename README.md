# EKS Authentication & Authorization Troubleshooting Lab

## Objective

Connect kubectl to an Amazon EKS cluster and troubleshoot authentication and authorization issues.

## Environment

- Amazon EKS
- AWS CLI v2
- kubectl v1.36
- IAM User: Cisco
- Region: us-east-2

---

## Problem

When attempting to access the EKS cluster:

```bash
kubectl get ns

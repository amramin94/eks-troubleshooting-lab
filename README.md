# EKS Authentication vs Authorization Troubleshooting

## Scenario

While connecting to an Amazon EKS cluster, kubectl was unable to access cluster resources.

## Initial Error

```bash
kubectl get ns
```

Output:

```text
You must be logged in to the server (the server has asked for the client to provide credentials)
```

## Troubleshooting Steps

Verified AWS identity:

```bash
aws sts get-caller-identity
```

Refreshed kubeconfig:

```bash
aws eks update-kubeconfig \
  --region us-east-2 \
  --name demo-eks-cluster
```

Verified EKS token generation:

```bash
aws eks get-token \
  --region us-east-2 \
  --cluster-name demo-eks-cluster
```

## Finding

Authentication to AWS was successful.

After further investigation:

```bash
kubectl get nodes
```

returned:

```text
Error from server (Forbidden):
User "arn:aws:iam::574070664760:user/Cisco"
cannot list resource "nodes"
```

## Root Cause

AWS authentication was working correctly, but Kubernetes RBAC permissions were insufficient.

## Key Learning

- Unauthorized = Authentication issue
- Forbidden = Authorization/RBAC issue
- EKS requires both IAM authentication and Kubernetes authorization

## Skills Practiced

- Amazon EKS
- Kubernetes RBAC
- AWS IAM
- kubectl troubleshooting
- AWS CLI
- Root Cause Analysis

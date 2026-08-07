# EKS LoadBalancer Service Without Subnet Tags

## Objective

Investigate whether Amazon EKS requires subnet tagging to provision an external LoadBalancer Service.

---

## Background

Many EKS tutorials and documentation recommend tagging subnets to allow Kubernetes and AWS Load Balancers to automatically discover suitable subnets.

Common tags include:

```text
Public Subnets:
kubernetes.io/role/elb = 1

Private Subnets:
kubernetes.io/role/internal-elb = 1
```

These tags are traditionally used by cloud providers and load balancer integrations to determine where AWS load balancers should be provisioned.

---

## Lab Setup

Environment:

```text
Amazon EKS
Kubernetes 1.36
AWS LoadBalancer Service
Default VPC
Public Subnets
```

No subnet tags were configured.

Verification:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<vpc-id>"
```

Result:

```text
No kubernetes.io/role/elb tags found.
No kubernetes.io/role/internal-elb tags found.
```

---

## Test

Created a LoadBalancer Service:

```bash
kubectl expose deployment nginx \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80 \
  -n demo-ns
```

Verification:

```bash
kubectl get svc -n demo-ns
```

Output:

```text
NAME    TYPE           EXTERNAL-IP
nginx   LoadBalancer   a29de113cbb1243859816dac553a1373.elb.amazonaws.com
```

AWS automatically provisioned an external Elastic Load Balancer.

---

## Observation

Despite the absence of subnet tags:

```text
kubernetes.io/role/elb
kubernetes.io/role/internal-elb
```

the LoadBalancer Service was created successfully.

AWS automatically discovered suitable public subnets and deployed the load balancer.

The application became reachable using the generated ELB endpoint.

---

## Validation

Check the service:

```bash
kubectl get svc -n demo-ns
```

Describe the service:

```bash
kubectl describe svc nginx -n demo-ns
```

Confirm AWS-created load balancers:

```bash
aws elb describe-load-balancers
```

---

## Key Finding

In this lab environment:

- No EKS subnet tags were configured.
- A public LoadBalancer Service was created successfully.
- AWS automatically selected public subnets.
- The application became reachable through a public ELB endpoint.

This indicates that modern EKS/AWS integrations can automatically discover suitable public subnets in some environments without requiring explicit subnet tagging.

---

## Key Takeaway

While subnet tagging is still considered a best practice and may be required in production environments or more complex network topologies, a basic EKS deployment in a default VPC may successfully provision external LoadBalancer Services without manually configured subnet tags.

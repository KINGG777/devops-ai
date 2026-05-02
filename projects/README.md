# Boutique Microservices — Deployment Guide

This guide walks through deploying the boutique e-commerce application from infrastructure provisioning to Kubernetes deployment with observability and logging.

---

# 📐 Architecture Overview

* Frontend → Gateway → Backend Services (Auth, Product, Order, User)
* PostgreSQL database
* Prometheus + Grafana for metrics
* CloudWatch for logs (via Fluent Bit)

---

# ⚠️ IMPORTANT PREREQUISITE

Before starting Terraform:

```bash
helm version
```

If Helm is NOT installed, install it first.

👉 Helm is REQUIRED because:

* ArgoCD is deployed using Helm
* Prometheus + Grafana are deployed using Helm

❗ Terraform does NOT install these components.

---

# 🐳 Local Development (Optional)

```bash
cd projects/boutique-microservices
docker-compose -f docker-compose.yml up -d
docker ps
```

Access:

* Frontend → http://localhost:3000
* Grafana → http://localhost:3007
* Prometheus → http://localhost:9090

Stop:

```bash
docker-compose -f docker-compose.yml down
```

---

# ☁️ Infrastructure Provisioning (Terraform)

```bash
cd projects/Infrastructure

terraform init
terraform plan
terraform apply --auto-approve
```

---

## ✅ What Terraform Creates

* VPC (public subnets)
* EKS Cluster
* Node Group (EC2)
* IAM roles
* ECR repositories

> ⚠️ Note: ArgoCD and Grafana are NOT created by Terraform. They are installed later using Helm.

---

# 🔗 Connect to Cluster

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name eks-cluster

kubectl get nodes
```

---

# 🚀 Deploy Application (Kubernetes)

```bash
kubectl apply -k gitops/
kubectl get pods -n boutique
```

---

# 🗄️ Restore Database

```bash
kubectl apply -f gitops/k8s/database/restore-job.yml

kubectl get pods -n boutique -l job-name=boutique-db-restore
kubectl logs -n boutique -l job-name=boutique-db-restore
```

---

# 🔄 Setup ArgoCD (GitOps)

```bash
kubectl apply -f gitops/argo-cd.yml -n argocd
kubectl get application -n argocd
```

Access UI:

```bash
kubectl port-forward svc/argocd-server 8443:443 -n argocd
```

Get password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

---

# 📊 Observability (Prometheus + Grafana)

Access:

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
kubectl port-forward svc/kube-prometheus-stack-grafana 8080:80 -n monitoring
```

* Prometheus → http://localhost:9090
* Grafana → http://localhost:8080

---

# 📦 Log Forwarding to CloudWatch (Fluent Bit)

Fluent Bit collects Kubernetes logs and sends them to CloudWatch.

---

## 🔧 Install Fluent Bit

```bash
MSYS_NO_PATHCONV=1 helm upgrade --install aws-for-fluent-bit aws/aws-for-fluent-bit \
  --namespace amazon-cloudwatch \
  --create-namespace \
  --set cloudWatch.enabled=true \
  --set cloudWatch.region=us-east-1 \
  --set cloudWatch.logGroupName=/eks/boutique/pods \
  --set cloudWatch.logStreamPrefix=from-fluent-bit-
```

---

## ✅ Verify

```bash
kubectl get pods -n amazon-cloudwatch
kubectl logs -n amazon-cloudwatch -l app.kubernetes.io/name=aws-for-fluent-bit
```

---

## 📁 CloudWatch Log Groups (Auto Created)

* `/eks/boutique/pods` → created by Fluent Bit
* `/aws/eks/fluentbit-cloudwatch/logs` → default Fluent Bit logs
* `/aws/lambda/*` → created automatically when Lambda runs

---

## ⚠️ Common Issue (Git Bash Path Bug)

Wrong log group created:

```
C:/Program Files/Git/eks/boutique/pods
```

✔ Fix:

```bash
MSYS_NO_PATHCONV=1
```

---

## 🔐 Required IAM Permissions

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "*"
}
```

---

## 🔁 Flow

Pods → Fluent Bit → CloudWatch → Log Group Auto Created

---

# 🔌 Port Forwarding

```bash
kubectl port-forward svc/frontend 3000:3000 -n boutique
kubectl port-forward svc/gateway 3001:3001 -n boutique
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
kubectl port-forward svc/kube-prometheus-stack-grafana 8080:80 -n monitoring
kubectl port-forward svc/argocd-server 8443:443 -n argocd
```

---

# 🔑 Credentials

## Grafana

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
-o jsonpath="{.data.admin-password}" | base64 --decode
```

Username: admin

---

## ArgoCD

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

Username: admin

---

# 🧹 Cleanup

```bash
cd projects/Infrastructure
terraform destroy --auto-approve
```

---

# ✅ Final Flow

Terraform → EKS → Deploy Apps → ArgoCD → Prometheus/Grafana → Fluent Bit → CloudWatch

---

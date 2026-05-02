# AIOps Assistant — Kira

An AI-powered SRE assistant built on AWS Bedrock Agent.
Kira diagnoses production incidents by querying:

* CloudWatch Logs
* CloudWatch Metrics (via Prometheus)
* EKS cluster health

…and responds with **root cause, evidence, and fix recommendations**. 

---

# 🏗️ Architecture

```text id="8o8p6q"
Streamlit UI (app.py)
      │
      ▼
Bedrock Agent (Kira)
      │
      ├── fetch_logs         → CloudWatch Logs
      ├── fetch_metrics      → Prometheus (ELB)
      └── fetch_service_health → EKS cluster
```

---

# ⚠️ IMPORTANT PREREQUISITES

Before starting:

* EKS cluster must be running
* Fluent Bit must be installed (logs → CloudWatch)
* Prometheus must be exposed via LoadBalancer
* AWS CLI must be configured

```bash id="i7y5g2"
aws configure
kubectl get pods -n amazon-cloudwatch
kubectl get svc -n monitoring
```

---

# ⚙️ Step 1: Setup IAM Roles

```bash id="3b0ywi"
chmod +x setup-iam.sh
./setup-iam.sh
```

Creates:

| Role                     | Purpose          |
| ------------------------ | ---------------- |
| aiops-lambda-role        | Lambda execution |
| aiops-bedrock-agent-role | Bedrock agent    |

---

# ⚙️ Step 2: Create Lambda Functions

Create 3 Lambda functions:

| Name                | Purpose            |
| ------------------- | ------------------ |
| aiops-fetch-logs    | CloudWatch logs    |
| aiops-fetch-metrics | Prometheus metrics |
| aiops-fetch-health  | EKS health         |

Settings:

* Runtime: Python 3.12
* Timeout: 30 seconds
* Memory: 512 MB (recommended)

---

# 🧪 Lambda Testing (IMPORTANT)

Always test before using AI.

---

## 🔹 Test Logs Lambda

```json id="2a8zsh"
{
  "parameters": [
    { "name": "filter_pattern", "value": "ERROR" },
    { "name": "log_group", "value": "/eks/boutique/pods" },
    { "name": "hours_back", "value": "1" },
    { "name": "region", "value": "us-east-1" }
  ]
}
```

---

## 🔹 Test Metrics Lambda

```json id="2r9hkw"
{
  "parameters": [
    { "name": "metric_name", "value": "pod_cpu_utilization" },
    { "name": "hours_back", "value": "1" }
  ]
}
```

---

## 🔹 Test Health Lambda

```json id="z38f0x"
{
  "parameters": [
    { "name": "cluster_name", "value": "eks-cluster" },
    { "name": "region", "value": "us-east-1" }
  ]
}
```

---

# ⚙️ Step 3: Configure Prometheus

```bash id="w8cy0j"
kubectl patch svc kube-prometheus-stack-prometheus -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc -n monitoring
```

Update in Lambda:

```python id="6a9ptp"
PROMETHEUS_URL = "http://<ELB>:9090"
```

---

# 🤖 Step 4: Deploy Bedrock Agent

```bash id="q9u5yb"
chmod +x deploy.sh
./deploy.sh
```

---

# 💻 Step 5: Run UI

```bash id="d6s0ap"
cp .env.example .env
```

Edit:

```env id="s8m3yx"
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<YOUR_AGENT_ID>
BEDROCK_AGENT_ALIAS_ID=TSTALIASID
```

Run:

```bash id="qpj5iw"
pip install -r requirements.txt
python -m streamlit run app.py
```

Open:

```id="1lpt0v"
http://localhost:8501
```

---

# 🧠 Example Queries

* check errors in last 1 hour
* is my cluster healthy
* CPU usage across services
* why service is failing

---

# ❌ Common Issues & Fixes

---

## 1. Lambda Timeout

```text id="7x4g3q"
Task timed out
```

✔ Fix:

* Increase timeout to 30 seconds
* Increase memory

---

## 2. No Logs Found

✔ Ensure:

* Fluent Bit running
* Log group exists: `/eks/boutique/pods`

---

## 3. CloudWatch Empty

✔ Fix:

* Check IAM role on node
* Restart Fluent Bit

```bash id="o9yqk1"
kubectl rollout restart daemonset aws-for-fluent-bit -n amazon-cloudwatch
```

---

## 4. Bedrock Error

```text id="v7h2kl"
dependencyFailedException
```

✔ Fix:

* Check Lambda logs
* Fix code issue

---

## 5. Prometheus Not Reachable

✔ Fix:

* Ensure ELB created
* Lambda not inside private VPC

---

## 6. Agent Stuck in PREPARING

✔ Wait 30–60 seconds or check schema/Lambda ARN

---

## 7. Wrong Log Group (Windows Issue)

```text id="p9yx8r"
C:/Program Files/Git/eks/boutique/pods
```

✔ Fix:

```bash id="6e5t3y"
MSYS_NO_PATHCONV=1
```

---

# 🔁 Final Flow

```text id="y0fq4v"
User → UI → Bedrock → Lambda → Logs/Metrics/Health → Response
```

---

# 🎯 Notes

* Logs come from `/eks/boutique/pods` (Fluent Bit)
* Lambda log groups auto-created on execution
* AI works only if logs + metrics are available

---

# 🚀 Result

You now have:

* AI-based root cause analysis
* Kubernetes observability
* Automated diagnostics system

👉 Production-ready AIOps pipeline


# Production-Grade Kubernetes Logging Pipeline (Fluent Bit + Loki + Grafana)

A production-aligned Kubernetes centralized logging pipeline deployed locally via OrbStack (ARM64), architected to mirror production cloud environments (AWS EKS + CloudWatch / Grafana Cloud).

## Architecture Overview

```text
+--------------------------------------------------------------+
| Kubernetes Cluster                                          |
|                                                              |
|  [ payment-service-simulator ]                               |
|        |                                                     |
|        v (stdout/stderr)                                     |
|  /var/log/pods/*/*/*.log (Node hostpath storage)             |
|        |                                                     |
|        v (mount & tail)                                      |
|  [ Fluent Bit (DaemonSet) ]                                  |
|        | -> Parser: cri                                      |
|        | -> Filter: kubernetes (Pod & Namespace metadata)     |
|        | -> Output: Loki HTTP API                            |
|        v                                                     |
|  [ Loki (Log Engine) ] <------- [ Grafana (Visualization) ]  |
+--------------------------------------------------------------+
```

## Tech Stack & Components

* **Container Runtime:** containerd (CRI log format)
* **Log Shipper:** Fluent Bit (DaemonSet with in-cluster service account)
* **Storage Backend:** Grafana Loki (Lightweight log aggregation system)
* **Dashboard / Visualization:** Grafana (LogQL querying)
* **Target Workload:** `payment-service-simulator` emitting both structured JSON events and legacy plain-text logs.

## Repository Structure

```text
k8s-logging-pipeline/
├── README.md
├── manifests/
│   ├── 00-namespace.yaml
│   └── 01-log-generator.yaml
└── helm/
    ├── fluent-bit-values-local.yaml
    └── loki-stack-values.yaml
```

## Deployment Steps

### 1. Provision Lab Namespace & Workload

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-log-generator.yaml
```

### 2. Deploy Centralized Log Storage (Loki Stack)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki-stack \
  --namespace logging-lab \
  -f helm/loki-stack-values.yaml
```

### 3. Deploy Log Shipper (Fluent Bit DaemonSet)

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm upgrade --install fluent-bit fluent/fluent-bit \
  --namespace logging-lab \
  -f helm/fluent-bit-values-local.yaml
```

## LogQL Validation Queries

Access Grafana via port-forwarding:

```bash
kubectl port-forward -n logging-lab svc/loki-grafana 3000:80
```

Credentials: `admin` / `admin`

* **Inspect Raw Streams:**
```logql
{job="fluent-bit"} |= "payment-service"
```

* **Filter Runtime Errors:**
```logql
{job="fluent-bit"} |= "ERROR"
```

* **Extract JSON Payloads & Filter by Attribute:**
```logql
{job="fluent-bit"} | json | amount > 250
```

## Next Phase (Cloud Production Migration)

แผนการพัฒนาในระยะถัดไป (Next Phase):
* **Terraform Infrastructure as Code (IaC):** เขียนโครงสร้าง Terraform เพื่อ Provision ทรัพยากรบน AWS แบบอัตโนมัติ:
  * **VPC & Networking:** ออกแบบ Public/Private Subnets สำหรับรองรับ EKS และ Node Groups
  * **EKS Cluster:** ติดตั้งและตั้งค่า Kubernetes Cluster บน AWS
  * **OIDC & IRSA (IAM Roles for Service Accounts):** กำหนดสิทธิ์แบบ Least Privilege เพื่อให้ Fluent Bit สามารถ Assume IAM Role ยิง Log เข้าสู่ AWS CloudWatch Logs / Grafana Cloud ได้อย่างปลอดภัยโดยไม่ต้องใช้ Static Credentials
* **Production Log Shipping:** นำ `helm/fluent-bit-values-cloudwatch.yaml` มาใช้งานจริงบน EKS พร้อมตั้งค่า Log Stream, Log Group และ Data Retention


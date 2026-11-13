# Cloud-Native 3-Tier DevSecOps Platform on AWS EKS

A production-oriented, automated DevSecOps pipeline and Kubernetes deployment architecture for a 3-tier web application (**React/HTML Frontend**, **Node.js Backend**, **MySQL Database**) hosted on **Amazon EKS** (`eu-west-3`).

Built to enforce shift-left security gates, secretless AWS authentication via **GitHub OIDC**, immutable **commit-SHA** container tagging, and cluster observability with **Prometheus & Grafana**.

---

## Architecture Overview

```text
Developer Push / PR (qa / main)
        │
        ▼
┌───────────────────────────────────────────────────────────────────┐
│               GitHub Actions (Self-Hosted WSL2 Runner)            │
│                                                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │
│  │  Gitleaks   │ │   Checkov   │ │    Trivy    │ │  SonarQube  │  │
│  │  (Secrets)  │ │ (IaC / K8s) │ │ (FS / Deps) │ │ (Code SAST) │  │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘  │
│         └───────────────┴───────┬───────┴───────────────┘         │
│                                 ▼                                 │
│             Docker Build & Immutable Commit-SHA Tagging           │
│                                 │                                 │
│                                 ▼                                 │
│              Push to Amazon ECR (GitHub OIDC Auth)                │
└─────────────────────────────────┬─────────────────────────────────┘
                                  │
                                  ▼
┌───────────────────────────────────────────────────────────────────┐
│                    Amazon EKS Cluster (eu-west-3)                 │
│                                                                   │
│   AWS ALB Ingress ──► Frontend / Node.js API Pods ──► MySQL Pod   │
│                                 │                                 │
│               Prometheus & Grafana (Cluster Metrics)              │
└───────────────────────────────────────────────────────────────────┘
```

---

## Key Engineering Highlights & Metrics

* **CI/CD Pipeline Optimization (`4m 22s` → `3m 43s`):** Migrated pipeline execution to a self-hosted Linux/WSL2 runner with persistent local tool caches and Docker layer reuse, cutting individual Trivy dependency scan times from **42s down to 11s**.
* **Parallel DAG Security Architecture:** Structured **Gitleaks**, **Checkov**, **Trivy**, and **SonarQube** as independent Directed Acyclic Graph (DAG) security gates ahead of container build and deployment stages—isolating failure gates while maintaining sequential queue stability on a single local runner.
* **Zero Long-Lived AWS Credentials (OIDC):** Configured **GitHub OpenID Connect (OIDC)** federation with scoped AWS IAM roles (`aws-actions/configure-aws-credentials@v4`), eliminating static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` secrets.
* **Immutable Image Traceability:** Tagged and pushed all container builds to **Amazon ECR** using `${{ github.sha }}` commit hashes instead of mutable `:latest` tags, enabling deterministic deployments and rapid rollbacks.
* **Environment Isolation & Ingress:** Separated `qa` and `prod` workloads across dedicated Kubernetes namespaces (`k8-manifests/qa` and `k8-manifests/prod`) routed via **AWS Application Load Balancer (ALB)**, **Route 53**, and **ACM** TLS certificates.

---

## DevSecOps Security Gates

| Stage | Tool | Scope | Security Control |
| :--- | :--- | :--- | :--- |
| **Secrets Detection** | **Gitleaks** | Full Git commit history & working tree | Prevents hardcoded credentials, API tokens, and private keys |
| **IaC & Manifest Audit** | **Checkov** | `Dockerfile` & `k8-manifests/` | Enforces CIS & Kubernetes security policies (`checkov.yaml`) |
| **SCA & Image Scanning** | **Trivy** | Client/Server dependencies & built Docker images | Identifies HIGH and CRITICAL CVEs with JSON & table reports |
| **Static Analysis (SAST)** | **SonarQube** | Application source code (`client/` & `server/`) | Detects code smells, reliability bugs, and security hotspots |

---

## Repository Structure

```text
devsecops-3tier-platform/
├── .github/workflows/
│   ├── qa-cicd.yml              # Shift-left security DAG, Docker build/scan, and QA EKS deploy
│   └── prod-cd.yaml             # Production release pipeline triggered on merge to main
├── client/                      # Frontend web application + Dockerfile
├── server/                      # Node.js backend API + Dockerfile
├── k8-manifests/
│   ├── qa/                      # QA namespace deployments, services, secrets & ALB ingress
│   └── prod/                    # Production namespace Kubernetes manifests
├── cluster-config.yaml          # Standard eksctl EKS cluster specification
├── free-tier-cluster.yaml       # Cost-optimized t3.small EKS cluster configuration
├── checkov.yaml                 # Custom Checkov policy rules
└── sonar-project.properties     # SonarQube scanner configuration
```

---

## Complete Setup & Deployment Guide

### 1. Prerequisites
Ensure the following CLI tools are installed and configured with AWS credentials (`aws configure`):
* `aws-cli` (v2+), `eksctl`, `kubectl`, `helm`, `docker`, and `git`

### 2. Create Amazon ECR Repositories
```bash
aws ecr create-repository --repository-name devsecops-client --region eu-west-3
aws ecr create-repository --repository-name devsecops-server --region eu-west-3
```

### 3. Provision the Amazon EKS Cluster
```bash
eksctl create cluster -f free-tier-cluster.yaml
aws eks update-kubeconfig --name devsecops-cluster --region eu-west-3
```

### 4. Configure IAM OIDC & Install AWS Load Balancer Controller
```bash
# Associate IAM OIDC provider with the EKS cluster
eksctl utils associate-iam-oidc-provider \
  --region eu-west-3 \
  --cluster devsecops-cluster \
  --approve

# Create IAM policy & service account for the ALB Controller
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy_v2.json

eksctl create iamserviceaccount \
  --cluster=devsecops-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region eu-west-3 \
  --approve

# Install AWS Load Balancer Controller via Helm
helm repo add eks [https://aws.github.io/eks-charts](https://aws.github.io/eks-charts)
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=devsecops-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 5. Create Namespaces, Database Secrets & Deploy Manifests
```bash
kubectl create namespace qa
kubectl create namespace prod

# Create MySQL credentials secret from template (never commit plain secrets)
cp k8-manifests/qa/mysql-secret.template.yaml k8-manifests/qa/mysql-secret.yaml
kubectl apply -f k8-manifests/qa/mysql-secret.yaml -n qa

# Deploy QA workloads and ALB Ingress
kubectl apply -f k8-manifests/qa/ -n qa
kubectl get ingress -n qa
```

### 6. Deploy Observability Stack (Prometheus & Grafana)
```bash
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo update
kubectl create namespace monitoring

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

---

## Complete Clean Teardown (Cost-Control Protocol)

Always delete Kubernetes Ingresses, Services, and Persistent Volume Claims **before** deleting the EKS cluster so AWS controllers cleanly remove external ALBs, Security Groups, and EBS disks:

```bash
# 1. Delete Ingresses & LoadBalancer Services (removes AWS ALBs & Target Groups)
kubectl delete ingress --all -A
kubectl delete svc --all -n qa --ignore-not-found
kubectl delete svc --all -n prod --ignore-not-found

# 2. Delete Persistent Volume Claims & Observability release
kubectl delete pvc --all -A --ignore-not-found
helm uninstall kube-prometheus-stack -n monitoring --ignore-not-found

# 3. Destroy the EKS cluster, nodegroups, and CloudFormation stacks
eksctl delete cluster --name devsecops-cluster --region eu-west-3 --wait
```

### Post-Teardown AWS Cost Verification
```bash
# Verify 0 active Load Balancers
aws elbv2 describe-load-balancers --region eu-west-3 --query "LoadBalancers[*].LoadBalancerName"

# Verify 0 unattached EBS volumes
aws ec2 describe-volumes --region eu-west-3 --filters Name=status,Values=available --query "Volumes[*].VolumeId"

# Verify 0 active NAT Gateways
aws ec2 describe-nat-gateways --region eu-west-3 --filter Name=state,Values=available --query "NatGatewayId"
```

---

## Author

**Gireesh Singh** — M.Sc. Computer Security Candidate @ EPITA, Paris  
* **GitHub:** [github.com/gireesh395](https://github.com/gireesh395)  
* **LinkedIn:** [linkedin.com/in/gireesh-singh395](https://linkedin.com/in/gireesh-singh395)
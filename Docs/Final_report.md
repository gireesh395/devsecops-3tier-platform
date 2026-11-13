# Building a Production-Grade 3-Tier DevSecOps Platform on AWS EKS: An Engineering Case Study & Post-Mortem

**Author:** Gireesh Singh  
**Repository:** `devsecops-3tier-platform`  
**Cloud Region:** AWS `eu-west-3` (Paris)  
**Core Stack:** AWS EKS, Amazon ECR, GitHub Actions (Self-Hosted WSL2), GitHub OIDC, AWS ALB Ingress, Gitleaks, SonarQube Cloud, Checkov, Trivy

---

## 1. Executive Summary & Project Vision

Most beginner cloud projects follow a fragile workflow: write code on a laptop, build a Docker image locally, tag it `:latest`, and manually run `kubectl apply` against a cluster. While that works for a quick demo, it collapses in real-world engineering environments because human error, leaked database passwords, vulnerable dependencies, and misconfigured infrastructure slip straight into production.

My goal with **devsecops-3tier-platform** was to engineer a realistic, automated **shift-left DevSecOps pipeline** from the ground up. Instead of treating security as a final checklist right before launch, I embedded automated security gates directly into every stage of the **GitHub Actions** CI/CD lifecycle.

**The Core Rule of the System:** No code can reach **Amazon ECR** (the container registry) or **AWS Elastic Kubernetes Service (EKS)** unless it automatically proves it has zero leaked secrets, clean code quality, hardened infrastructure manifests, and zero critical package vulnerabilities.

---

## 2. System Architecture & Pipeline Flow

The platform follows a classic **3-tier microservices pattern** (a frontend user interface, a Node.js/Express REST API backend, and a MySQL relational database) isolated into separate `qa` and `prod` Kubernetes namespaces on a single AWS EKS cluster.

### End-to-End Execution Flow
1. **Code Push (`qa` or `main`):** A developer pushes code or merges a pull request, triggering the self-hosted GitHub Actions runner on WSL2.
2. **Shift-Left Security Gates (Blocking):**
   - **Gitleaks** scans commit history for hardcoded secrets or credentials.
   - **SonarQube Cloud** performs Static Application Security Testing (SAST) and enforces a strict Quality Gate.
   - **Checkov** audits the `Dockerfile` and Kubernetes manifests (`k8s-manifests/`) for security misconfigurations.
   - **Trivy** scans application dependencies (filesystem) and compiled container images for known CVEs.
3. **Immutable Container Build & Push:** Once all security gates pass, Docker builds the container image, tags it with the exact **40-character Git commit SHA**, and pushes it to **Amazon ECR** using temporary **GitHub OIDC** credentials (zero static AWS keys).
4. **Automated EKS Deployment:** The CD pipeline updates the Kubernetes Deployment manifest with the new commit-SHA image tag and rolls it out to either the `qa` or `prod` namespace on **AWS EKS (`eu-west-3`)**, routed externally through an **AWS Application Load Balancer (ALB) Ingress**.

---

## 3. Why I Chose Each Tool (Deep-Dive Logic, Project Examples & Industry Context)

Choosing tools for a DevSecOps pipeline is not about collecting trendy logos—every tool in this stack solves a specific failure mode across code, credentials, infrastructure, or runtime operations.

---

### 1. AWS Elastic Kubernetes Service — EKS (`eu-west-3`)
* **The Engineering Logic:** Manually SSH-ing into virtual machines to start Node.js or MySQL processes fails the moment a process crashes or traffic spikes. Kubernetes acts as a declarative control loop: you define the desired state in YAML (`1` healthy API pod, `1` MySQL pod), and EKS continuously reconciles reality to match that state.
* **Concrete Example in My Project:** In my `prod` namespace, `nodejs-app` communicates with the database simply by connecting to the internal hostname `mysql` on port `3306`. Even when a pod restarts and AWS assigns it a brand-new internal IP address, Kubernetes CoreDNS automatically updates the DNS record behind `service/mysql` so the application never loses its database connection. Deploying both `qa` and `prod` namespaces on a single cluster in Paris (`eu-west-3`) gave me realistic multi-environment isolation without paying for two separate EKS control planes (~$73/month each).
* **Real-World Industry Scenario:** Companies like **Spotify** or **Airbnb** run hundreds of microservices across isolated namespaces on Kubernetes so dozens of engineering teams can deploy independently without stepping on each other's networking, secrets, or CPU limits.

---

### 2. GitHub Actions (Self-Hosted WSL2 Runner)
* **The Engineering Logic:** Standard GitHub-hosted cloud runners are ephemeral virtual machines—every time you push code, GitHub boots a clean VM, downloads system packages from scratch, pulls vulnerability databases over the network, and destroys the VM when the job finishes. That "cold-start setup time" quietly eats up minutes on every single commit.
* **Concrete Example in My Project:** Running Trivy and Checkov repeatedly on cloud runners meant re-downloading Trivy's binary and CVE database on every push. By hosting my own runner on Ubuntu/WSL2 and adding an idempotency check (`if ! command -v trivy &> /dev/null`), the runner installs tools once and reuses its local disk cache across subsequent runs—cutting setup overhead down to seconds. *(Note: Because self-hosted runners retain state and access internal credentials, they should strictly be used on private or controlled repositories—never on public repos accepting untrusted pull requests from forks.)*
* **Real-World Industry Scenario:** Engineering teams with heavy Docker builds, large `node_modules` trees, or private VPC-only staging clusters run self-hosted runners inside their own AWS VPC so CI pipelines can deploy directly to private Kubernetes API endpoints without exposing them to the public internet.

---

### 3. GitHub OIDC + AWS IAM Roles (Zero Static Secrets)
* **The Engineering Logic:** Traditional CI/CD tutorials tell beginners to create an IAM User, generate permanent `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` strings, and paste them into GitHub Secrets. Those static keys are like leaving a spare house key under the doormat: they never expire, and if a malicious dependency or log leak exposes them, an attacker has permanent access to your AWS account. **OpenID Connect (OIDC)** replaces static keys with a cryptographic handshake—like a hotel keycard programmed to open one specific room and expire automatically after 15 minutes.
* **Concrete Example in My Project:** When `prod-cd.yaml` runs, GitHub's OIDC provider (`token.actions.githubusercontent.com`) mints a short-lived signed JWT token containing claims proving the job came from `repo:gireesh395/devsecops-3tier-platform`. AWS Security Token Service (STS) verifies that token against my IAM Role's Trust Policy and issues temporary credentials valid only for pushing to Amazon ECR and updating my EKS cluster. Zero long-lived AWS keys exist anywhere in my repository or GitHub settings.
* **Real-World Industry Scenario:** Enterprise security teams enforce OIDC across all cloud deployments because it satisfies strict compliance audits (SOC 2, ISO 27001), eliminates manual 90-day key rotation chores, and logs the exact GitHub repository and branch name inside **AWS CloudTrail** for every single API call.

---

### 4. Amazon Elastic Container Registry — ECR (`eu-west-3`)
* **The Engineering Logic:** Pulling container images from public registries like Docker Hub during automated deployments introduces two major risks: strict anonymous pull rate limits (`429 Too Many Requests` errors that break deployments mid-rollout) and slower cross-internet downloads.
* **Concrete Example in My Project:** Initially, my pipeline pushed images to Docker Hub (`3d52835`), which created unnecessary external dependencies and public exposure. Migrating to **Amazon ECR** in `eu-west-3` (`c277648`) placed my private container images in the exact same AWS region as my EKS worker nodes. Node pulls stay inside AWS's high-speed regional network, and each image is stored with its immutable **40-character Git commit SHA** tag so I always know the exact code running in `prod`.
* **Real-World Industry Scenario:** Financial and healthcare organizations store all production container images in private regional registries (ECR, Google Artifact Registry) with immutable tag policies enabled—guaranteeing no engineer or attacker can silently overwrite an existing image tag in production.

---

### 5. AWS Application Load Balancer (ALB) Ingress Controller
* **The Engineering Logic:** In Kubernetes, setting `type: LoadBalancer` on every `Service` provisions a separate, billable cloud load balancer (~$18–$25/month each) with its own public IP. Even worse, it exposes internal services directly to the internet. An **Ingress Controller** works like a single front-desk receptionist for an entire office building: external visitors arrive at one main entrance, and the receptionist routes them internally to the right department.
* **Concrete Example in My Project:** Instead of exposing both the frontend/API (`port 5000`) and MySQL (`port 3306`) publicly, I configured both as internal `ClusterIP` services (`bfadd75`) and deployed a single **AWS ALB Ingress** (`app-ingress.yaml`) listening on HTTP port `80` (`6edc338`). External users hit `k8s-prod-appingre-*.eu-west-3.elb.amazonaws.com`, which routes traffic cleanly to `nodejs-service`, while `mysql` remains 100% unreachable from outside the cluster.
* **Real-World Industry Scenario:** SaaS platforms running 50+ microservices behind a single domain (`api.example.com/auth`, `api.example.com/billing`, `api.example.com/orders`) use an ALB Ingress to terminate TLS/HTTPS certificates in one central place, attach AWS WAF firewall rules, and route paths to dozens of private `ClusterIP` services.

---

### 6. Gitleaks (Automated Secret Detection)
* **The Engineering Logic:** Once a database password, AWS key, or JWT secret is committed to Git, simply deleting it in the next commit does **not** remove it—it stays permanently readable in `.git` commit history. Automated scrapers continuously scan public GitHub commits and compromise leaked credentials within seconds.
* **Concrete Example in My Project:** Gitleaks runs as the very first gate in my CI pipeline, scanning commit diffs before any Docker build starts. Alongside `.gitignore` rules that block live `mysql-secret.yaml` files (allowing only `mysql-secret.template.yaml` blueprints), Gitleaks acts as a safety net ensuring my MySQL credentials or SonarQube tokens can never accidentally be pushed.
* **Real-World Industry Scenario:** Open-source maintainers and enterprise DevSecOps teams run Gitleaks both locally as a `pre-commit` hook (blocking the commit on the developer's laptop before it even leaves their machine) and in CI/CD to prevent accidental credential leaks across hundreds of repositories.

---

### 7. SonarQube Cloud (SAST & Supply-Chain Quality Gate)
* **The Engineering Logic:** Human code reviews catch high-level design flaws, but reviewers easily miss subtle injection bugs, unhandled promise rejections, duplicated logic, or risky CI/CD workflow definitions. Static Application Security Testing (SAST) analyzes source code and configuration files *without executing them* to catch flaws early.
* **Concrete Example in My Project:** Integrating SonarQube Cloud (`a21d7bd`) immediately paid off in a real, tangible way: even after my deployment jobs turned green, SonarQube flagged `.github/workflows/prod-cd.yaml` with a **Security Rating: C** because third-party GitHub Actions were referenced by mutable version tags (`uses: actions/checkout@v4`) rather than immutable 40-character commit SHAs. It caught a supply-chain risk in my pipeline code that functional testing completely missed.
* **Real-World Industry Scenario:** Enterprise engineering organizations configure GitHub Branch Protection Rules to block Pull Request merges automatically whenever SonarQube reports a failed Quality Gate—preventing technical debt and OWASP Top 10 vulnerabilities (like SQL Injection or Cross-Site Scripting) from ever entering `main`.

---

### 8. Checkov (Infrastructure-as-Code & Kubernetes Policy Auditing)
* **The Engineering Logic:** Modern cloud breaches rarely happen because someone broke AES-256 encryption—they happen because an engineer forgot one line in a YAML file, leaving a container running as `root`, exposing a storage bucket publicly, or omitting health checks. Checkov scans `Dockerfile` and Kubernetes manifests against hundreds of CIS Benchmark security policies *before* those resources are ever created.
* **Concrete Example in My Project:** When I converted Checkov into a hard-blocking gate (`71d3750`), it flagged my `Dockerfile` for lacking a container `HEALTHCHECK` instruction. Fixing that (`c6df401`) ensured Docker and Kubernetes can detect if the Node.js event loop hangs and automatically restart the container, while `.checkov.yaml` codified an intentional security baseline for my single-cluster setup.
* **Real-World Industry Scenario:** Cloud platform teams use Checkov (or Open Policy Agent / Kyverno) to block developers from deploying privileged pods (`privileged: true`), containers without CPU/memory limits (which can crash an entire shared worker node), or unencrypted AWS EBS volumes.

---

### 9. Trivy (Filesystem & Container Image CVE Scanner)
* **The Engineering Logic:** Even if every line of JavaScript you write is 100% bug-free, a standard Node.js application pulls in hundreds of open-source npm packages and sits on top of a Linux base image (`node:18-alpine` or `debian`) containing dozens of OS libraries. If OpenSSL or `express` has a critical Remote Code Execution vulnerability (CVE), your container inherits it.
* **Concrete Example in My Project:** Trivy scans both my application's filesystem dependencies (`package-lock.json`) and the compiled Docker image *before* `docker push` sends it to Amazon ECR. If a `CRITICAL` or `HIGH` CVE is detected, the pipeline aborts immediately—guaranteeing that only vetted, patched container images ever enter my private registry.
* **Real-World Industry Scenario:** After high-profile supply-chain incidents like **Log4Shell (CVE-2021-44228)**, organizations embedded Trivy into CI/CD pipelines and nightly registry scans so they could instantly identify every container running a vulnerable library and block unpatched builds from reaching production.

## 4. Core Architectural Logic & FinOps Sizing

### A. Immutable Commit-SHA Image Tagging vs. `:latest`
Tagging every new build as `myapp:latest` makes production rollbacks risky because there is no deterministic link between the running container and the exact source commit that built it.

To solve this, my pipeline tags every Docker image with its exact **Git commit SHA** (e.g., `4733842`, `1e07740`) and automatically patches the Kubernetes Deployment manifest during the CD stage. Every single pod running in EKS can be traced back to the exact commit that produced it.

### B. Zero-Trust Kubernetes Secret Management
Database passwords are never hardcoded inside Kubernetes YAML files or Git repositories:
1. Live secret files are blocked via `.gitignore` so sensitive values never enter version control—only sanitized `*.template.yaml` blueprints are tracked in Git.
2. At runtime, the Node.js backend pod reads its MySQL host, username, and password dynamically from Kubernetes Secrets using `secretKeyRef`, keeping credentials completely decoupled from application code.

### C. Cloud FinOps & Compute Sizing Logic (Why `t3.micro` + 1 Pod Replica?)
Running an AWS EKS cluster can quickly become expensive if oversized nodes (`m5.large` or `t3.medium`) are left running continuously. I engineered this cluster around **FinOps cost-efficiency** while deliberately solving the technical constraints of small worker nodes:

1. **Why `t3.micro` Worker Nodes (2 vCPUs, 1 GiB RAM):**
   - **Cost Control:** `t3.micro` instances (~$0.0104/hr) are AWS Free Tier eligible and allow me to run a real managed Kubernetes cluster in `eu-west-3` at minimal cost.
   - **Burstable CPU Credits:** AWS `T3` instances accumulate CPU credits during quiet periods and burst up to full core speed during CI/CD deployments and container startups—ideal for spiky development workloads.
2. **Why `replicas: 1` (Instead of Multi-Pod Scaling):**
   - **Memory Overhead Math:** Out of a `t3.micro`'s `1 GiB` (1024 MiB) of RAM, Kubernetes system daemons (`kubelet`, `kube-proxy`, `aws-node` VPC CNI, and `CoreDNS`) reserve a baseline share, leaving roughly **600–700 MiB** of allocatable memory for application pods.
   - **AWS VPC CNI IP Limits:** On AWS EKS, every pod receives a real VPC IP address via Elastic Network Interfaces (ENIs). Smaller EC2 instances have a strict ceiling on how many ENIs and secondary IPv4 addresses they can attach (`t3.micro` supports only 2 ENIs x 2 IPs = **4 max pods** by default).
   - **The Engineering Trade-Off:** When I initially scaled `nodejs-app` to `replicas: 2`, the second replica hit node memory and IP scheduling ceilings alongside the MySQL pod. Setting `replicas: 1` for both `qa` and `prod`—paired with a `Recreate` deployment strategy—keeps the cluster completely stable without memory exhaustion or IP starvation, while keeping the manifests ready to scale horizontally on larger production nodes.

### D. Branch Isolation (`qa` vs. `main`) & Cost-Safe Teardown
- **Environment Promotion:** Feature work and integration testing happen on the `qa` branch (deploying to the `qa` Kubernetes namespace). Only tested, verified changes are merged into `main` to trigger the production workflow (`prod-cd.yaml`) into the `prod` namespace.
- **Controlled Teardown:** To avoid idle cloud charges when not actively testing, cluster configurations are consolidated cleanly in `infra/eks-cluster.yaml` so the entire EKS infrastructure can be spun up or torn down deterministically.

---

## 5. Deep-Dive Troubleshooting: 6 Real Roadblocks & Breakthroughs

### 1. Resolving Internal Service Discovery (`ENOTFOUND`) & Ingress Exposure
- **The Problem:** During initial deployments, the Node.js backend crashed immediately with `ENOTFOUND` database connection errors, while external browser requests could not reach the frontend cleanly.
- **Root Cause & Logic:** In Kubernetes, pods are ephemeral—every time a container restarts, it receives a brand-new internal IP address. Hardcoding pod IPs breaks immediately. Additionally, giving every tier its own AWS `LoadBalancer` wastes cloud resources and exposes backend services unnecessarily.
- **The Solution:**
  - Deployed a dedicated internal Kubernetes `ClusterIP` Service (`service/mysql` on port `3306`) in front of the MySQL pod so Kubernetes CoreDNS reliably resolves `mysql` to the live database container.
  - Switched the application service to `ClusterIP` (`port 5000`) and placed an **AWS ALB Ingress** in front of it on HTTP port `80`, ensuring only the web entry point faces the public internet.
- **What I Learned:** Always separate internal service-to-service networking (`ClusterIP` + internal DNS) from external user traffic (`Ingress`).

### 2. Eliminating Static AWS Keys with GitHub OIDC Authentication
- **The Problem:** When connecting GitHub Actions to Amazon ECR and AWS EKS, AWS IAM repeatedly rejected authentication requests with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.
- **Root Cause & Logic:** OIDC authentication relies on a cryptographic handshake: GitHub mints a short-lived JSON Web Token (JWT) containing claims (`aud` for audience and `sub` for repository/branch identity), and AWS IAM validates those claims against a strict Trust Policy. Inspecting the logs revealed two mismatches: the workflow job lacked permission to request a token, and the IAM trust policy's `sub` condition did not match the repository claim format.
- **The Solution:**
  - Added explicit `permissions: id-token: write` to the deployment job so GitHub's runner could request an OIDC token.
  - Aligned the token audience to `sts.amazonaws.com` and configured the AWS IAM Trust Policy `StringLike` condition to explicitly trust `repo:gireesh395/devsecops-3tier-platform:*`.
- **What I Learned:** Zero-trust cloud automation requires exact claim alignment between the identity provider (GitHub) and verifier (AWS IAM), completely eliminating the security risk of leaked static AWS credentials.

### 3. Enforcing a Blocking Checkov IaC Gate Without False-Positive Gridlock
- **The Problem:** After consolidating Checkov into a hard-blocking security gate, every CI run failed immediately on `Dockerfile` and Kubernetes manifest checks.
- **Root Cause & Logic:** IaC scanners enforce dozens of strict enterprise rules out of the box. Rather than disabling Checkov—which defeats shift-left security—I inspected the failure logs to separate genuine container hardening gaps from intentional single-cluster lab settings.
- **The Solution:**
  - Hardened the `Dockerfile` by adding a proper `HEALTHCHECK` instruction so Docker and Kubernetes can actively monitor application responsiveness.
  - Created a version-controlled `.checkov.yaml` policy baseline to enforce critical security controls while cleanly documenting intentional exceptions.
- **What I Learned:** Effective DevSecOps balances strict automated enforcement with documented policy baselines—fixing real weaknesses rather than bypassing gates.

### 4. Solving Single-Node Resource Deadlocks During Kubernetes Rollouts
- **The Problem:** When deploying a new commit-SHA image to EKS, pods intermittently got stuck in a `Pending` state (`Insufficient cpu` / `Insufficient memory`), stalling the CD pipeline.
- **Root Cause & Logic:** Kubernetes' default `RollingUpdate` strategy spins up the **new** pod *before* terminating the **old** pod to achieve zero downtime. On a single `t3.micro` worker node, temporarily running two copies of the application stack simultaneously exceeded the node's available RAM and ENI IP capacity.
- **The Solution:**
  - Set pod replicas to `1` per environment to match the node's memory ceiling.
  - Switched the Kubernetes Deployment strategy from `RollingUpdate` to **`Recreate`**, instructing Kubernetes to gracefully terminate the old container and release its memory *before* scheduling the newly built commit-SHA image.
- **What I Learned:** Deployment strategies must match underlying compute capacity; `RollingUpdate` requires spare cluster headroom, whereas `Recreate` prevents resource deadlocks on constrained nodes.

### 5. Optimizing Self-Hosted Runner Caching & Trivy Installation
- **The Problem:** Running Trivy vulnerability scans on every commit initially added unnecessary build latency and occasional shell syntax errors during repeated `apt-get` installations on the self-hosted runner.
- **Root Cause & Logic:** Unlike ephemeral GitHub-hosted runners that wipe their disk after each job, a self-hosted WSL2 runner retains installed packages across runs. Re-downloading GPG keys and reinstalling Trivy on every commit wasted time and introduced avoidable network failure points.
- **The Solution:** Added an idempotency guard (`if ! command -v trivy &> /dev/null`) in the workflow step so Trivy installs cleanly once and executes immediately from local cache on all subsequent runs.
- **What I Learned:** Writing idempotent CI scripts ("check if a tool is already installed before installing") makes self-hosted runners significantly faster and more reliable.

### 6. Supply-Chain Hardening: Pinning GitHub Actions to Full Commit SHAs
- **The Problem:** Even after all functional CI/CD jobs passed, **SonarQube Cloud** flagged `.github/workflows/prod-cd.yaml` with a **Security Rating: C** (`Use full commit SHA hash for this dependency`).
- **Root Cause & Logic:** Referencing third-party GitHub Actions via mutable version tags (such as `uses: actions/checkout@v4`) creates a supply-chain vulnerability: if an upstream repository is compromised, the `v4` tag can be reassigned to malicious code without changing your workflow file.
- **The Solution:** Audited all `uses:` dependencies across `.github/workflows/` to pin every third-party action to its immutable **40-character Git commit SHA** (e.g., `actions/checkout@<40-char-sha> # v4`), bringing the CI/CD pipeline itself up to the same immutability standard as the Docker images.
- **What I Learned:** True DevSecOps secures the **delivery pipeline itself**, not just the application code traveling through it.

---

## 6. Final Verification & System Outcome

The completed platform operates end-to-end across `qa` and `main` (`prod`) branches:
- **Automated Security Gates:** Every push passes **Gitleaks**, **SonarQube Cloud**, **Checkov**, and **Trivy** before building an immutable commit-SHA Docker image into **Amazon ECR**.
- **Live AWS EKS Production Stack:** Both `mysql` and `nodejs-app` pods run healthy (`1/1 Running`, `0` restarts) inside the `prod` namespace on **AWS EKS (`eu-west-3`)**, serving live HTTP traffic via an active **AWS ALB Ingress** (`*.eu-west-3.elb.amazonaws.com`).
- **Visual Documentation:** Full architecture diagrams and live deployment proof screenshots are documented in `README.md` and `assets/`.
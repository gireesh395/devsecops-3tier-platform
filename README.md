# DevSecOps 3-Tier Platform (QA on AWS EKS)

This repository contains Kubernetes manifests and CI/CD workflows for deploying a 3-tier Node.js application backed by MySQL on AWS EKS.

## Architecture

- **Application:** Node.js Express server (`nodejs-app`) running on port `5000`.
- **Database:** MySQL 8.0 (`mysql`) running internally on port `3306`.
- **Networking:** Exposed externally via an AWS LoadBalancer Service on port `5000`.

## Incident Log & Fixes

1. **ImagePullBackOff:** Resolved by creating the `regcred` Docker registry secret in the `qa` namespace.
2. **CrashLoopBackOff (`ENOTFOUND mysql`):** Resolved by deploying the internal `mysql` Deployment and Service to provide CoreDNS name resolution for the Node.js backend.
3. **External Connectivity:** Updated `nodejs-service` to `type: LoadBalancer` to provision an AWS public load balancer.
